# Changelog

All notable user-visible changes to the File Browser TOS application package.

## 2.32.0-2 — 2026-11-20

Rectification of the 2.32.0-1 App Center listing review (on-hold notices for
x86_64 and aarch64). Each review code is addressed below.

### Added
- **C3 (privacy policy missing):** `PRIVACY.md` describes what the package
  stores, what leaves the NAS, third parties, retention, user rights, security
  measures and the deletion channel. `help` in `config.ini` and in
  `tosapp.json` now points at it, so the App Center entry reaches the policy
  directly.
- **F22 (declared path without `webui.bz2`):** the package ships
  `src/webui.bz2`, a bzip2-compressed tar holding a one-file `index.html` that
  forwards to `http://<current-host>:8128/` (the app's declared access path).
  `scripts/build.sh` stages it and `tools/verify.py` validates it; the official
  self-test tool accepts the extra top-level entry with its D20 warning.
- Repository layout section and full build/verify/run instructions in `README.md`.

### Changed
- **F2 (container fails to start on aarch64):** the compose file pinned only the
  arguments (`command: ["-p", ...]`) and relied on the image's own `ENTRYPOINT`
  metadata, which the reviewer's aarch64 runtime did not get: docker exec'd the
  first argument and failed with
  `exec: "-p": executable file not found in $PATH`. The service now pins
  `entrypoint: ["/filebrowser"]` to the binary the image ships and keeps the
  flags as `command`, so the service starts on x86_64 and aarch64 alike.
- **C9 (description inconsistent with the delivered functionality):** the
  `zh-cn` and `zh-hk` descriptions were carried over from an unrelated
  application (habit tracking); they now describe File Browser, and the English
  description is limited to features the package actually delivers (file
  management, share links, users/permissions, search).
- **C10 (repository / directory compliance):** the README is complete (what the
  package is, how it runs, file manifest, build, verification, store submission,
  limitations, privacy) and the repository layout is documented; `PRIVACY.md`
  sits next to `README.md` as the review expects.
- **F11 (residual configuration/data after uninstall):** no uninstall hook can
  be shipped in a Docker application package, and the App Center keeps
  `/Volume1/DockerAppData/<appid>/` on uninstall by platform policy. The data
  location and the "delete the directory to remove everything" step are now
  documented in `README.md`, `PRIVACY.md` §8 and the `important` field.
- **Health check** tightened on the same service (`timeout` 5s, `start_period`
  20s) and still probes the UI on container port 8080.
- `.lang` re-rendered from `tosapp.json` (23 languages, 14 required covered).
- `tosapp.json` metadata cleaned: the stale `probe` block (left over from another
  application) and the unused `entrypoint_file` field were removed.

### Known limitations
- **T1 (high/critical CVE in the upstream image):** upstream has released
  v2.63.23, but that image changed layout completely (`tini` + `/init.sh`,
  `/config`, `/database`, `/srv`, environment-driven configuration) and needs
  its own packaging migration, including a database path change for existing
  installations. This packaging iteration keeps the pinned, reviewed
  `filebrowser/filebrowser:v2.32.0` image and does not claim the CVE items as
  fixed; the migration is tracked as a separate version.
- Uninstalling the application leaves `/Volume1/DockerAppData/filebrowserdocker/`
  in place (platform directory policy); delete it manually to remove all data.

## 2.32.0-1 — 2026-11-14

### Added
- First release. File Browser (`filebrowser/filebrowser:v2.32.0`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8128 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/filebrowserdocker/`.
