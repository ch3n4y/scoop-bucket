# scoop-ch3n4y

Personal [Scoop](https://scoop.sh) bucket for software I use that is not available in the official buckets.

## Install the bucket

```powershell
scoop bucket add ch3n4y https://github.com/ch3n4y/scoop-ch3n4y
```

The bucket name is `ch3n4y`, so a package from here is installed as `scoop install ch3n4y/<app>`.
A plain `scoop install <app>` works too, as long as no other bucket has a package with the same name.

## Packages

| Package | Upstream | Description |
| --- | --- | --- |
| `pail` | [DejavuMoe/Pail](https://github.com/DejavuMoe/Pail) | Desktop client for S3-compatible object storage (S3, R2, MinIO, Wasabi, VersityGW). Ships `pail-desktop` and the `pail` CLI. |
| `easytier-manager` | [ch3n4y/easytier-manager](https://github.com/ch3n4y/easytier-manager) | Desktop manager for [EasyTier](https://github.com/EasyTier/EasyTier) mesh VPN. Windows x64 only. |
| `aisafetest` | [ch3n4y/AISafeTest](https://github.com/ch3n4y/AISafeTest) | AgSec agent security evaluation platform. Releases come from cnb.cool, not GitHub. |

```powershell
scoop install ch3n4y/pail
```

Installs a Start menu shortcut `Pail` plus two shims:

- `pail-desktop` — the desktop app
- `pail` — the command-line tool (shares the desktop app's accounts and sync jobs)

### easytier-manager

```powershell
scoop install ch3n4y/easytier-manager
```

The upstream Windows build is an NSIS installer whose payload is a single self-contained
Tauri binary. This manifest unpacks it instead of running it, so the app installs green —
no uninstall registry entry and nothing for the installer to leave behind.

The binary declares `requestedExecutionLevel level="requireAdministrator"`, so **every launch
raises a UAC prompt**; EasyTier needs admin rights to manage its virtual adapter and routes.
Launch it from the Start menu, or from a shell with `sudo easytier-manager`.

Settings are stored in `%APPDATA%\com.ch3n4y.easytier-manager` and survive `scoop update`.
The app has its own Tauri updater pointed at its GitHub releases, independent of Scoop.

### aisafetest

```powershell
scoop install ch3n4y/aisafetest
```

**Releases live on cnb.cool, not GitHub.** The `AISafeTest` GitHub repo stops at v1.0.1, while the
authoritative feed is the `ch3n4y/AiSafeTool-Releases` project on cnb.cool (currently v2.6.48).
Each release ships per-platform zips, a `SHA256SUMS` list and an `aisafetest-latest.json` feed. This
manifest reads that feed as the single source of truth for both the version and the hashes:

```json
"checkver": {
    "url": "https://cnb.cool/ch3n4y/AiSafeTool-Releases/-/git/raw/main/aisafetest-latest.json",
    "jsonpath": "$.tag_name",
    "regex": "v([\\d.]+)"
}
```

`autoupdate` picks each architecture's hash from `$.assets[?(@.name == '...')].sha256` in the same
feed, matching on the full asset name. The `github` checkver helper is unusable here because these
tags exist only on cnb.cool.

The archive holds `AISafeTest.exe` plus `AISafeTest.sig`; the plain executable name is what `bin`
and `shortcuts` reference. The app has its own signed updater, independent of Scoop.

## Notes

- Upstream builds are unsigned. SmartScreen may warn on first launch; choose **More info → Run anyway**.
- Executables inside the archives carry the version and architecture in their filename
   (`pail-desktop-v0.2.0-windows-amd64.exe`). A `pre_install` step renames them to plain
   `pail-desktop.exe` / `pail.exe`, because Scoop resolves `bin` and `shortcuts` paths literally —
   it does not expand `$version` or `$architecture` there. The plain names keep working across
   version bumps without touching `autoupdate`.
- `checkver` follows the upstream GitHub releases and `autoupdate` verifies downloads against the
  release's `SHA256SUMS` file.

## Adding a package

1. Put a manifest in `bucket/<name>.json` following the
   [Scoop app manifest reference](https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests).
2. Validate it:

   ```powershell
   scoop install bucket/<name>     # or
   scoop checkup
   ```

3. Generate `autoupdate` with `scoop shim`-free tooling such as
   [excavator](https://github.com/ScoopInstaller/Excavator) or keep it manual and bump by hand.

## License

Manifests are provided under the upstream project's license (MIT for Pail). Package binaries are not
redistributed here; Scoop downloads them from the upstream releases.
