# core stack

The foundational self-hosted services that power the local AI briefing pipeline.

## Services

### RSSHub (`port 1200`)
Converts non-RSS sources into RSS feeds. Here it's used to expose Twitter/X list
timelines as RSS, which the briefing script polls directly. Requires Twitter auth
credentials via environment variables (see `.env.example`).

Backed by Redis for response caching so repeated fetches don't hit rate limits.
Cache TTL is set to 30 minutes (`CACHE_EXPIRE=1800`) to avoid timeouts when the
briefing script hits feeds that haven't been warmed recently.

### Redis (`127.0.0.1:6379`)
Cache layer for RSSHub and the ai-briefing Telegram bot. Bound to `127.0.0.1:6379`
so host-side scripts (`briefing.py`, `bot.py`) can reach it via `localhost` without
the IP changing on container restarts. Data is persisted to `/mnt/docker/core/redis`.

### FreshRSS (`port 8080`)
Web-based RSS reader and aggregator. Optional — useful for browsing raw feeds
manually before they're processed by the briefing pipeline. Data and extensions
are persisted to `/mnt/docker/core/freshrss`.

Feed auto-refresh is enabled via `CRON_MIN=*/15` (every 15 minutes).

### KOReader Sync Server / kosync (`port 7200`)
Self-hosted reading-progress sync server for KOReader devices (e-readers, phones,
tablets). Bundles its own Redis instance internally for state storage. Data and
logs persist to `/mnt/docker/core/kosync`. Reachable on the LAN and over the
NordVPN meshnet (`100.93.206.138:7200`) — port `7200/tcp` must be allowlisted in
NordVPN's own firewall (`nordvpn allowlist add port 7200`) since it blocks
inbound traffic by default, the same way port `8083` is allowlisted for calibre-web.

The image internally listens on two ports: `7200` for HTTPS (self-signed cert)
and `17200` for plain HTTP. KOReader devices are configured with a plain
`http://` sync URL, so the compose mapping is `"7200:17200"` — host port `7200`
forwards to the container's plain-HTTP listener, not its TLS one. Connecting to
host port 7200 over plain HTTP works as a result; the container's own TLS port
is not exposed to the host.

### Calibre-web (`port 8083`)
Web UI and OPDS catalog for browsing/downloading the epub library. Library files
live on the `/mnt/media` drive (`/mnt/media/books`) rather than the `/mnt/docker`
drive, since it's bulk content storage, not container state — matches how
`/mnt/media` is already used for `movies/`/`tv/`. App config (settings, user
accounts, `metadata.db` if a fresh library is initialized) persists to
`/mnt/docker/core/calibre-web/config`. Includes the `universal-calibre`
DOCKER_MOD for ebook-convert support (format conversion, cover/metadata
extraction) — remove the `DOCKER_MODS` env var if that's not needed.

Reachable on the LAN and over the NordVPN meshnet (`100.93.206.138:8083`); port
`8083/tcp` is allowlisted via `nordvpn allowlist add port 8083` the same way as
kosync's `7200`.

First-run setup wizard requires an existing `metadata.db` at the library path —
it does not offer to create one itself, and will reject an empty directory with
a vague "New db location is invalid" error. Bootstrap an empty library first
with the bundled `calibredb` CLI (from the universal-calibre mod) before
running the wizard:
```
docker exec -u 1000:1000 calibre-web calibredb list --with-library /books
```
Then enter `/books` (the container path, not `/mnt/media/books`) in the wizard.

KOReader's OPDS catalog browser can point at `http://<host>:8083/opds` to browse
and download books directly from this library; reading progress for those books
then syncs separately via [kosync](#koreader-sync-server--kosync-port-7200).

### Nextcloud (behind Caddy on `port 443`) + nextcloud-db
Self-hosted file sync/storage with WebDAV support. Backed by a dedicated MariaDB
container (`nextcloud-db`) rather than SQLite, per Nextcloud's own recommendation
for anything beyond throwaway testing. DB credentials are set via
`NEXTCLOUD_DB_ROOT_PASSWORD`/`NEXTCLOUD_DB_PASSWORD` in `.env`; admin account is
bootstrapped on first run via `NEXTCLOUD_ADMIN_USER`/`NEXTCLOUD_ADMIN_PASSWORD`
(only takes effect while `/var/www/html` is empty).

App code/config lives on the SSD (`/mnt/docker/core/nextcloud/html`,
`/mnt/docker/core/nextcloud/db`), but actual user file storage is bind-mounted
from the `/mnt/storage` drive (`/mnt/storage/nextcloud/data` →
`/var/www/html/data` in the container) since it's bulk content, not container
state — same reasoning as `/mnt/media/books` for calibre-web.

Not published directly to the host — only reachable through the `caddy`
reverse proxy on the `core_default` network (`nextcloud:80`). The Nextcloud
mobile apps (Android/iOS) hard-refuse plain HTTP ("Strict Mode: No HTTP
connection allowed!"), so `overwriteprotocol=https` and
`trusted_proxies=172.18.0.0/16` (the `core_default` subnet) are set via `occ`
so Nextcloud generates correct URLs and trusts Caddy's forwarded headers.

Reachable via two URLs for the same instance/account — LAN IP and NordVPN
meshnet IP — both listed in `NEXTCLOUD_TRUSTED_DOMAINS`. Note that env var is
only read by the entrypoint on first install; adding a domain to an
already-installed instance (e.g. the meshnet IP, added after initial setup)
requires `occ config:system:set trusted_domains <index> --value=<domain>`
directly, or it 400s with `{"error": "Trusted domain error.", "code": 15}`.

### Caddy (`port 443`)
TLS-terminating reverse proxy in front of Nextcloud. Uses Caddy's internal CA
to self-sign certs (`tls internal`) for the LAN IP (`192.168.1.50`), the
NordVPN meshnet IP (`100.93.206.138`), and the `homelab` hostname — no public
domain needed. Since none of those are real SNI-capable hostnames, clients
connecting by bare IP send no SNI at all, so `default_sni 192.168.1.50` is set
in the global options block so Caddy knows which site to serve by default.
Config is the bind-mounted `/mnt/docker/core/caddy/Caddyfile`; cert/key
material persists to `/mnt/docker/core/caddy/data`.

Self-signed means each client must manually trust the cert on first connect
(browsers: click through the warning once; mobile apps: accept the cert
prompt). Reachable on the LAN and over the NordVPN meshnet
(`https://100.93.206.138`); port `443/tcp` is allowlisted via
`nordvpn allowlist add port 443` the same way as kosync/calibre-web.

WebDAV root: `https://192.168.1.50/remote.php/dav/files/<username>/`
(or `https://100.93.206.138/remote.php/dav/files/<username>/` over meshnet).

## How it fits together

```
Twitter/X lists
      │
      ▼
   RSSHub :1200  ←──  Redis :6379 (localhost)
      │                     ▲           │
      ▼                     │           ▼
 briefing.py  ──────── stores items   bot.py reads items
      │                               on button press
      ▼
 Claude CLI  →  Telegram
```

## Setup

1. Copy `.env.example` to `.env` and fill in your Twitter credentials.
2. `docker compose up -d`

## Environment variables

See `.env.example` for required variables. Never commit `.env`.
