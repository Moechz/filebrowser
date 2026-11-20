# Privacy Policy — File Browser for TerraMaster TOS 7

This policy describes how the **File Browser** Docker application package
(`filebrowserdocker`) handles data when installed on your TerraMaster NAS.

- Package: `filebrowserdocker` (TOS 7 App Center, Docker application)
- Upstream software: [File Browser](https://github.com/filebrowser/filebrowser) 2.32.0
- Publisher: Moechz

## 1. Scope

The application runs the official `filebrowser/filebrowser` Docker Hub image,
unmodified, as a non-root container (uid 1000). This policy covers the package
and its default configuration. Users, share links and settings you create inside
File Browser, and any reverse proxy or external integration you add yourself,
follow their own behaviour.

## 2. Data stored on your NAS

Everything File Browser needs is stored locally, under
`/Volume1/DockerAppData/filebrowserdocker/`:

| Data | Where | Why |
|---|---|---|
| File Browser database (users, hashed passwords, permissions, settings) | `data/filebrowser.db` | Authentication and configuration |
| Files and folders you manage or upload through the web UI | `data/` | Core functionality — this directory *is* your file storage |
| Share links and their expiry / password hashes | `data/filebrowser.db` | Sharing |
| Preview thumbnails and temporary upload fragments | `data/` (inside the container's writable area) | Previews and resumable uploads |

Nothing in this list is uploaded to the publisher or to TerraMaster. The
publisher operates no server that receives data from this application.

## 3. Data sent off your NAS

By default the application is fully offline: it serves the web UI on
`http://<NAS-IP>:8128/` on your local network and sends no telemetry, analytics
or crash reports.

Outbound connections happen only in these cases:

- **Image download / update** — when the App Center installs or updates the
  package, Docker pulls `filebrowser/filebrowser:v2.32.0` from Docker Hub
  (or from the registry mirror configured in DockerEngine). This is a standard
  image pull and carries no user data.
- **Links you create or open** — if you share a file link with someone and they
  open it, their browser connects to *your* NAS; nothing goes to the publisher.
- **Optional integrations you configure** — a reverse proxy, external
  authentication provider, or a mount pointing at cloud storage you set up
  yourself. These are your own choices and are not part of this package.

The application does **not** contact upstream project servers, does not check
for updates on its own, and contains no advertising or tracking code.

## 4. Third parties

| Recipient | Data | Purpose |
|---|---|---|
| Docker Hub (or your configured registry mirror) | Image pull request metadata (IP address, image name/tag) | Install and update the container image |
| Nobody else by default | — | — |

No data is sold or shared for advertising. The publisher does not receive any of
the above. File Browser has no built-in cloud service, no account system outside
your NAS, and no third-party login.

## 5. Retention

Data is kept for as long as the application is installed and you keep it:

- Files you upload stay until you delete them in the web UI or remove the data
  directory.
- The database keeps user accounts, settings and share links until you delete
  them in the UI (Settings → User management / Share management) or remove the
  data directory.
- Logs are written to the container's stdout/stderr only and are discarded with
  the container; the Docker logging driver keeps them according to its own
  rotation settings.

Uninstalling the application does **not** delete
`/Volume1/DockerAppData/filebrowserdocker/` — see §8.

## 6. Your rights

You control the data directly on the NAS:

- **Access / portability** — download your files through the web UI; the SQLite
  database (`data/filebrowser.db`) can be copied for a full export.
- **Rectification** — rename, move or edit files and user accounts in the UI.
- **Erasure** — delete files, users and share links in the UI, or delete the
  data directory to remove everything.
- **Restriction** — do not create share links, and keep the service on the LAN
  behind your firewall.

Because the publisher never receives your data, there is nothing to request from
the publisher; all requests are handled locally by you.

## 7. Security measures

- The container runs as uid 1000 (non-root), without privileged mode, host
  networking, added capabilities, or access to the Docker socket.
- The database stores password hashes (bcrypt), not plain-text passwords. The
  first-run account is `admin` / `admin` — change that password immediately.
- Data lives in `/Volume1/DockerAppData/filebrowserdocker/`, owned by the
  application user, and is not exposed to other NAS users by the package.
- The web UI is served over plain HTTP on your LAN. If you expose it to the
  internet, put it behind HTTPS (reverse proxy) and enable authentication
  (for example TOS's own authentication or a proxy auth provider).

## 8. Deletion channel

To delete all data: App Center → uninstall the application, **then** delete
`/Volume1/DockerAppData/filebrowserdocker/` (for example with File Manager or
`rm -rf`). The uninstall step removes the container and the package files; the
data directory is kept by the platform's directory policy so that a reinstall
keeps your files. There is no server-side copy to delete.

## 9. Changes

Material changes to this policy are published in this file and noted in the
package changelog with the version that introduced them.

## 10. Contact

Open an issue at <https://github.com/Moechz/filebrowser/issues>.
