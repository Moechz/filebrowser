# filebrowser-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [File Browser](https://github.com/filebrowser/filebrowser).

## What this is

A TOS Docker application archive (`filebrowserdocker.tar.gz`) that installs
File Browser on TOS 7 through the App Center. The archive contains the four
files the official Docker spec requires, plus the TOS front-end bundle the App
Center review asks for (F22):

- `config.ini` — TOS application metadata (`application_type: docker`, id `filebrowserdocker`)
- `filebrowserdocker.lang` — 23-language superset (TOS requires 14)
- `filebrowserdocker.svg` — application icon (346 bytes, 4 SVG elements)
- `docker-compose.yml` — one service, image `filebrowser/filebrowser:v2.32.0`
- `webui.bz2` — bzip2-compressed tar with a one-file `index.html` that forwards
  to `http://<current-host>:8128/`, the access path declared by `config.ini`

## Repository layout

```text
.
├── README.md                     this file
├── PRIVACY.md                    privacy policy (the package help link points here)
├── CHANGELOG.md                  released versions and review remediation
├── LICENSE                       Apache-2.0 (upstream licence)
├── tosapp.json                   build spec — single source of truth
├── TOSAppSelfTestingTool         official spec checker (shipped by the toolkit)
├── src/                          files copied into the package
│   ├── config.ini
│   ├── filebrowserdocker.lang
│   ├── filebrowserdocker.svg
│   ├── docker-compose.yml
│   └── webui.bz2                 TOS front-end bundle (F22)
├── scripts/
│   ├── build.sh                  stage → verify → pack one architecture
│   └── build-dual.sh             build x86_64 + aarch64 and run the official check
├── tools/
│   └── verify.py                 review-standards self-check (hard gate)
├── test/
│   └── install-and-verify.sh     on-device E2E (needs root; not run by CI)
└── out/                          build products (git-ignored, published as Release assets)
```

## How it works

| | |
|---|---|
| Container image | `filebrowser/filebrowser:v2.32.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8128/` (container port 8080) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/data` → `/Volume1/DockerAppData/filebrowserdocker/data` |
| Logs | `docker logs filebrowserdocker` (stdout/stderr) |

## Design highlights

- **Explicit start command (F2).** The service pins
  `entrypoint: ["/filebrowser"]` and passes its flags as `command`
  (`-p 8080 -r /data -d /data/filebrowser.db`). It does not rely on the image's
  own `ENTRYPOINT` metadata: the 2.32.0-1 review found an aarch64 runtime where
  that metadata was missing, docker exec'd the first argument and the container
  died with `exec: "-p": executable file not found in $PATH`.
- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 8080 (`timeout` 5s, `start_period` 20s).
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check below.
- **TOS front-end bundle (F22).** `src/webui.bz2` is a 338-byte bzip2-compressed
  tar containing `index.html` (a `location.replace()` forward to port 8128). The
  official self-test tool accepts the extra top-level entry with a
  `D20` warning; `config.ini` declares the matching access path.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/filebrowserdocker/data/filebrowser.db` | Users, password hashes, settings, share links | first start | small (KB–MB) | persistent |
| `/Volume1/DockerAppData/filebrowserdocker/data/` | Your files and folders, previews and temporary upload fragments | on demand | bounded by your own data | persistent |

## Data location and uninstall (F11)

All data lives under `/Volume1/DockerAppData/filebrowserdocker/`:

- **Upgrade / restart** — the data directory is kept, so files and accounts
  survive (`docker compose` recreates only the container).
- **Uninstall** — the App Center removes the container and the package files but
  **keeps** `/Volume1/DockerAppData/filebrowserdocker/`, by platform directory
  policy (the same policy that lets a reinstall keep your data). No uninstall
  hook can be shipped inside a Docker application package.
- **Delete everything** — uninstall, then delete
  `/Volume1/DockerAppData/filebrowserdocker/` yourself (File Manager or
  `rm -rf`). This removes the files, the database, the users and the share links.

Back up that directory before uninstalling if you may want the data again.

## Privacy

The package is offline by default: no telemetry, no analytics, no crash
reporting, and no contact with upstream project servers. Data stays on the NAS;
only the Docker image pull touches the network. The full policy — what is
stored, what leaves the NAS, third parties, retention, your rights, security
measures and the deletion channel — is in [PRIVACY.md](PRIVACY.md), and it is
also reachable from the App Center: the `help` field of `config.ini` points at
<https://github.com/Moechz/filebrowser/blob/main/PRIVACY.md>.

## Upstream image and CVE status (T1)

The 2.32.0-1 review recorded a high/critical CVE finding (T1) against the
upstream image and asked for a component upgrade.

This version keeps `filebrowser/filebrowser:v2.32.0`. Upstream has since
released newer images, but from v2.33.0 on the image is a different packaging:
`tini` + `/init.sh`, configuration under `/config`, the database under
`/database`, file roots under `/srv`, and no `/filebrowser` entrypoint. Moving
to it is a packaging migration with a database path change for existing
installations, tracked as a separate version — it is **not** part of 2.32.0-2,
and this version does not claim the T1 items as fixed. What this version does
offer against the CVE item is a container that is actually running (F2) and a
package that no longer relies on image metadata that differs per architecture.

## Build

```bash
scripts/build.sh                  # -> out/filebrowserdocker.tar.gz + .sha256
scripts/build.sh 2.32.0-2         # packaging iteration bump
scripts/build.sh 2.32.0-2 aarch64
scripts/build-dual.sh 2.32.0-2    # both architectures + official check
```

The build stages the package files (`webui.bz2` included when present),
substitutes the version/platform placeholders, removes CRLF/BOM/AppleDouble
(binary bundles such as `webui.bz2` are excluded from that pass), then runs
`tools/verify.py`, a review-standards self-check: archive layout (four required
files plus optional `webui.bz2`), `webui.bz2` is a bzip2 tar with `index.html`,
JSON validity, required fields, 14+ language coverage, version consistency,
Docker-Hub-only images, fixed tag, reserved/recommended host ports, non-root
user, per-service healthcheck/TZ/restart, a start command that cannot exec an
option, data path below the application data root, never the `/Volume*`
wildcard, comment-free sequence blocks, `x-app-meta` position, no literal
secrets and the SVG size/element limits.

## Local verification

```bash
./TOSAppSelfTestingTool -c out/filebrowserdocker_x86_64.tar.gz   # official spec check
./TOSAppSelfTestingTool -c out/filebrowserdocker_aarch64.tar.gz  # (D20 warning for webui.bz2 is expected)
bash test/install-and-verify.sh                                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README + PRIVACY only).
2. Release **tag `v2.32.0-2`**, assets `filebrowserdocker_x86_64.tar.gz` +
   `filebrowserdocker_aarch64.tar.gz` (and their `.sha256`).
3. Developer platform → application ID `filebrowserdocker`, package type Docker,
   repository URL, both architectures `x86_64` and `aarch64`.
4. Version Management → Add Version `2.32.0-2` → automated validation → review.

## License / branding

File Browser is distributed by its upstream authors under **Apache-2.0**. The container image is the official image from Docker Hub (`filebrowser/filebrowser:v2.32.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- The application runs inside a container; data lives in the application data directory (`/Volume1/DockerAppData/filebrowserdocker/`) as described above.
- The web UI is plain HTTP on the LAN; use a reverse proxy with TLS if you expose it.
- The image is pinned to v2.32.0; see the CVE status section above for the
  migration plan to the current upstream image.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
