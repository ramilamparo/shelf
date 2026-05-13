# book-reader

Self-hosted ebook + audiobook server. Books live in Google Drive, sync one-way to local bind mounts, served by Kavita (ebooks) and Audiobookshelf (audiobooks). Exposed via Cloudflare Tunnel — no port forwarding, no certbot.

## Stack

- **Kavita** — EPUB/PDF web reader, native cross-device progress sync
- **Audiobookshelf** — audiobook web app with streaming + range seek
- **rclone** — one-way `gdrive: → ./data/` sync loop, every 15 min
- **cloudflared** — outbound tunnel to Cloudflare edge, TLS terminated at edge

## Layout

```
.
├── docker-compose.yml
├── .env                       # gitignored, copy from .env.example
├── rclone/rclone.conf         # gitignored, generated once
├── config/                    # gitignored, app SQLite + caches
│   ├── kavita/
│   ├── abs-config/
│   └── abs-metadata/
└── data/                      # gitignored, rclone sync targets
    ├── books/
    └── audiobooks/
```

## Google Drive layout

```
MyDrive/
├── Books/         # EPUB / PDF, flat or nested
└── Audiobooks/    # one folder per book (M4B or per-chapter MP3)
```

## One-time setup

### 1. Authorize rclone for Google Drive

`rclone` is not needed on the host — generate the config via a throwaway container:

```sh
docker run --rm -it \
  --user "$(id -u):$(id -g)" \
  -v "$PWD/rclone:/config/rclone" \
  rclone/rclone:latest config
```

The `--user` flag is important — without it the container runs as root and the resulting `rclone.conf` ends up `root:root`, unreadable by the sync container.

In the interactive menu:
- `n` (new remote) → name: `gdrive` (must match `GDRIVE_REMOTE` in `.env`)
- Storage: `drive`
- `client_id` / `client_secret`: leave blank (uses rclone's default; fine for personal use)
- Scope: `2` (read-only) — server only reads from Drive
- Service account, advanced: defaults
- Auto-config: `n` (headless) → it prints a `rclone authorize` command to run on a machine with a browser; paste the resulting token back
- Team drive: `n`
- Confirm, then `q`

Verify:

```sh
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$PWD/rclone:/config/rclone:ro" \
  rclone/rclone:latest lsd gdrive:
```

You should see `Books` and `Audiobooks` listed.

### 2. Create Cloudflare Tunnel

In Cloudflare dashboard → **Zero Trust** → **Networks** → **Tunnels** → **Create a tunnel**:

1. Connector: **Cloudflared**
2. Name the tunnel (e.g. `home-books`)
3. Copy the **token** shown on the "Install connector" page — paste into `.env` as `TUNNEL_TOKEN`
4. Skip the install step (Docker handles it)
5. **Public Hostname** tab → add two:
   - `books.yourdomain.com` → Service: `HTTP` → URL: `kavita:5000`
   - `audio.yourdomain.com` → Service: `HTTP` → URL: `audiobookshelf:80`

### 3. Configure `.env`

```sh
cp .env.example .env
# edit .env: TUNNEL_TOKEN, PUID/PGID (run `id -u` / `id -g`), TZ
```

## Run

```sh
docker compose up -d
docker compose logs -f rclone        # watch first sync
```

First Drive sync can take hours for a large library (750 GB/day Drive quota).

Open `https://books.yourdomain.com` and `https://audio.yourdomain.com`.

For local testing before exposing publicly, the apps are also reachable at:
- Kavita: http://127.0.0.1:5000
- Audiobookshelf: http://127.0.0.1:13378

## Inside each app — first run

**Kavita** (http://127.0.0.1:5000):
1. Create admin user.
2. **Library** → Add Library → type `Book` → folder `/books`.
3. Settings → **Tasks** → Library Scan: set to e.g. every 30 min (not real-time — file watcher conflicts with rclone churn).

**Audiobookshelf** (http://127.0.0.1:13378):
1. Create root user.
2. **Settings** → **Libraries** → New Library → folder `/audiobooks`.
3. In the library settings, **disable the folder watcher**. Use **Settings → Scheduled Tasks** to run a library scan every 30 min instead.

## Useful commands

```sh
# force a manual sync now
docker compose exec rclone rclone sync gdrive:Books /data/books --drive-skip-gdocs

# tail logs
docker compose logs -f rclone
docker compose logs -f cloudflared

# stop everything
docker compose down

# pull updates
docker compose pull && docker compose up -d
```

## Notes & caveats

- **Read-only mounts**: reader containers see `data/` as `:ro`. Only the `rclone` container writes. Defense in depth — if a reader app misbehaves, it can't corrupt the synced library.
- **Cloudflare 100 MB cap**: applies only to HTTP request bodies (uploads). Streaming + downloads through the tunnel have no documented cap.
- **Cloudflare media TOS**: technically restricts large media from non-Cloudflare origins. Personal-scale audiobook streaming is low-risk but not officially blessed.
- **Drive duplicate filenames**: Drive allows two files with the same name in one folder, which can break ABS scans. Periodically run `docker compose exec rclone rclone dedupe gdrive:Audiobooks` if you hit weird scan results.
- **No backups**: this setup is a *reader*, not a backup. Drive is the source of truth. `config/` is local-only — back it up separately if you care about user accounts and read progress.
