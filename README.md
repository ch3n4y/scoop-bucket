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

```powershell
scoop install ch3n4y/pail
```

Installs a Start menu shortcut `Pail` plus two shims:

- `pail-desktop` — the desktop app
- `pail` — the command-line tool (shares the desktop app's accounts and sync jobs)

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
