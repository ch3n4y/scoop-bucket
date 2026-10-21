# AGENTS.md

Instructions for AI agents working in this repository.

This is a personal [Scoop](https://scoop.sh) bucket. Each package is one JSON manifest in `bucket/`.
There is no build system, no tests to run, and no code — the manifests *are* the product.

## Repository layout

```
bucket/<app>.json     one manifest per package; the filename is the package name
README.md             short, user-facing: what this bucket is and how to install from it
.vscode/settings.json JSON schema for editors
```

Keep it that way. Do not create subdirectories, scripts, or CI unless explicitly asked.

## README rules

`README.md` is deliberately short: a one-line description, the `scoop bucket add` command, a package
table with a single-sentence description each, `scoop install ch3n4y/<package>`, and `scoop update`.

- Add exactly one row to the package table when adding a package.
- Keep each description to one short sentence. Do not add install notes, UAC warnings, or
  implementation details — those belong in the manifest's `notes` field, not the README.
- Do not expand the README into documentation. If you feel the urge to explain internals, that
  explanation belongs here in AGENTS.md or in a comment inside the manifest.

## Adding a package

### 1. Find the real release source first

Do not assume GitHub Releases is authoritative. Check the project's own release tooling:

```bash
gh release list --repo <owner>/<repo>
gh api repos/<owner>/<repo>/contents/.github/workflows/release.yml
```

If a project publishes elsewhere (cnb.cool, a self-hosted feed, an installer CDN), the manifest must
point at that source. `aisafetest` is the worked example: GitHub is stale at v1.0.1 while cnb.cool
carries v2.6.48.

### 2. Inspect the actual artifact before writing the manifest

Download the Windows artifact and look inside. Never guess its structure.

```powershell
# checksum against whatever the upstream publishes (release digest, SHA256SUMS, feed JSON)
Get-FileHash <file> -Algorithm SHA256
# archive contents
[System.IO.Compression.ZipFile]::OpenRead(<zip>).Entries | ForEach-Object { $_.FullName }
# NSIS/installer contents, if applicable
7z l <setup.exe>
```

Then confirm the executable actually runs:

```powershell
$p = Start-Process <exe> -PassThru; Start-Sleep 8; $p.Refresh()
$p.HasExited; $p.MainWindowTitle      # then Stop-Process
```

Also read the embedded manifest — it decides whether every launch raises UAC:

```powershell
$t = [System.Text.Encoding]::UTF8.GetString([IO.File]::ReadAllBytes(<exe>))
[regex]::Matches($t, 'requestedExecutionLevel[^>]*').Value
```

`requireAdministrator` means every launch elevates. Put that in `notes`. `asInvoker` means normal
launches are fine even if the app elevates later for specific actions.

### 3. Pick the manifest shape

Choose the simplest shape that works:

| Situation | Approach |
| --- | --- |
| Portable exe/zip | plain `url` + `hash`, `bin` pointing at the exe |
| Exe filename contains the version | add `#/<fixed-name>` to the url so `bin` stays stable |
| Archive where filenames carry version/arch | `pre_install` renames to fixed names (see `pail`) |
| Real installer with a self-contained payload | unpack it with `installer.script` + `Expand-7zipArchive` (see `easytier-manager`) |
| Upstream publishes a JSON release feed | `checkver`/`hash` via `jsonpath` (see `aisafetest`) |

### 4. Hard rule: `bin` and `shortcuts` paths are literal

Scoop resolves the path strings inside `bin` and `shortcuts` **literally**. Only `$dir`,
`$original_dir` and `$persist_dir` are substituted there — there is no `$version` and no
`$architecture`. Confirmed in `lib/install.ps1` (`create_shims`) and `substitute` in `lib/core.ps1`.

So this fails at install time with `Can't shim ...: File doesn't exist.`:

```json
"bin": [["app-v$version-windows-$architecture.exe", "app"]]
```

Do note the distinction: `bin` and `shortcuts` *may* be placed inside `architecture.<arch>` —
`arch_specific` in `lib/manifest.ps1` supports that, so each architecture can list different files.
What does not work is putting placeholders **inside** a path string.

Use one of:

- `#/fixed.exe` appended to the url (best when there is one artifact per architecture)
- `pre_install` / `installer.script` renaming to a fixed name (best for multi-file archives)
- `architecture`-scoped `bin`/`shortcuts` entries with fully literal filenames

### 5. Provide `checkver` and `autoupdate`

Every manifest must have both, or `scoop update` cannot follow upstream.

```json
"checkver": { "github": "https://github.com/<owner>/<repo>" },
"autoupdate": {
    "architecture": {
        "64bit": { "url": "https://.../app-$version-windows-amd64.zip" }
    }
}
```

When upstream publishes a machine-readable feed, prefer `jsonpath` over HTML scraping:

```json
"checkver": {
    "url": "<feed-url>",
    "jsonpath": "$.tag_name",
    "regex": "v([\\d.]+)"
}
```

If archives differ per architecture, put the `hash` block **inside each architecture entry**, not at
the top level of `autoupdate` — a single top-level jsonpath would match both architectures and
cannot tell them apart.

### 6. Verify the install end to end

Publishing is not done until the package installs from the bucket. Push, then:

```powershell
scoop update
scoop install ch3n4y/<app>
```

Then check all of:
- The app directory contains only the expected payload plus Scoop metadata. An unexpected
  `*-setup.exe` or `$PLUGINSDIR` left behind means the extraction step did not do what you think.
- The shim exists in `D:\scoop\shims` and the executable runs through it.
- The Start menu shortcut exists, if the manifest declares one.

**A successful-looking install can be a failed one.** Exit code 0 and "installed successfully" print
even when an `installer.script` line throws, because Scoop continues past it. Always inspect the
installed directory, and read the install output for PowerShell errors rather than trusting the
success banner.

## Package notes

Facts worth knowing before touching these manifests.

- **pail** — archives carry version and architecture in the executable names, so `pre_install`
  renames them to `pail-desktop.exe` / `pail.exe`. Binaries are unsigned.
- **easytier-manager** — upstream ships only an NSIS installer (no portable zip). The payload is one
  self-contained Tauri binary, so `installer.script` unpacks it instead of running it. The exe
  declares `requireAdministrator`: every launch raises UAC. Windows x64 only.
- **aisafetest** — releases live on cnb.cool (`ch3n4y/AiSafeTool-Releases`), not GitHub. The
  authoritative feed is `/-/git/raw/main/aisafetest-latest.json`. Note the cnb.cool raw path form:
  `/-/git/raw/<branch>/<file>` works, `/-/raw/<branch>/<file>` returns 404.
- **wifit3** — single PyInstaller exe, renamed via `#/wifit3.exe`. Windows x64 only; no arm64 build
  upstream. The app binds supported USB adapters to WinUSB on first use (one UAC prompt), which
  detaches the normal driver. Upstream behaviours belong in `notes`, not in the README.

## Conventions

- Two-space or four-space JSON indentation — match the surrounding file.
- `description` is one sentence, ending with a period.
- `license` uses an SPDX identifier. Use `"Unknown"` when upstream declares no license.
- `homepage` is the project page, which is not necessarily the release source.
- Omit keys rather than setting them to empty values.

## Commits

One logical change per commit, imperative mood: `Add <app> manifest`, `Fix <app> shim names`,
`Update <app> to <version>`. Do not bundle a rename of the repository with manifest edits.
