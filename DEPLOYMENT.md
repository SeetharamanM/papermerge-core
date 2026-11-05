# Deploying Papermerge DMS

This guide deploys Papermerge as a complete, production-ready web application
from this repository using Docker Compose. Everything lives in [`deploy/`](./deploy):

| File | Purpose |
|------|---------|
| `deploy/docker-compose.yml` | Web app, OCR worker, PostgreSQL, Redis, optional HTTPS proxy |
| `deploy/.env.example` | All settings; copy to `deploy/.env` |
| `deploy/Caddyfile` | Automatic Let's Encrypt TLS (optional `https` profile) |

## What gets deployed

```
            browser
               │  http(s)
        ┌──────▼───────┐   (optional) caddy :443 ── automatic TLS
        │     web      │   nginx :80
        │  React UI    │   ├─ /            → React UI (built from ./ui)
        │  REST API    │   ├─ /api, /ws    → FastAPI/Django core (:8000)
        │  auth server │   └─ login pages  → auth server (:4010) + token validator
        └──┬───────┬───┘
           │       │  Celery tasks via Redis
   ┌───────▼─┐  ┌──▼────┐   ┌────────┐
   │postgres │  │ redis │◄──┤ worker │  OCR (Tesseract / OCRmyPDF)
   └─────────┘  └───────┘   └────────┘
   volumes: postgres_data, media (documents), index_db (search index)
```

The `web` image is built from [`docker/prod/Dockerfile`](./docker/prod/Dockerfile):
it compiles the React UI in `ui/`, installs the Python backend, and bundles the
upstream `papermerge/auth-server` and `papermerge/tivoli` (token validator)
components behind nginx. On start it runs migrations, creates the admin user
and applies the search index schema automatically.

> The Ember frontend in the separate `papermerge.js` repository targets the
> older 2.x API. Papermerge 3.x ships its own React UI (in `ui/`), and that UI
> is what this deployment serves. You do not need to deploy `papermerge.js`.

## Requirements

* A Linux host (VM, VPS, home server) with **2 vCPU / 2 GB RAM** minimum
  (OCR is CPU heavy; 4 GB+ recommended), x86_64 (`linux/amd64`)
* Docker Engine 24+ with the Compose plugin (`docker compose version`)
* ~5 GB disk for images, plus room for your documents
* For HTTPS: a domain name whose DNS A/AAAA record points at the host, and
  ports 80 and 443 open

Install Docker on Ubuntu/Debian:

```bash
curl -fsSL https://get.docker.com | sh
```

```bash
sudo usermod -aG docker $USER && newgrp docker
```

## 1. Get the code

```bash
git clone https://github.com/seetharamanm/papermerge-core.git
```

```bash
cd papermerge-core/deploy
```

## 2. Configure

```bash
cp .env.example .env
```

Generate a secret key and passwords:

```bash
openssl rand -hex 32
```

Edit `.env` and set at least:

| Variable | Description |
|----------|-------------|
| `SECRET_KEY` | Long random string (used to sign login tokens) |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` / `ADMIN_EMAIL` | Initial superuser, created on first start |
| `DB_PASSWORD` | PostgreSQL password (letters and digits only) |
| `OCR_LANGUAGES` | Tesseract languages installed in the image, e.g. `"deu fra spa"` (English is always included) |
| `OCR_DEFAULT_LANGUAGE` | Default OCR language, e.g. `eng` |
| `HTTP_PORT` | Port for the web UI (default `12000`) |
| `TIMEZONE` | e.g. `Asia/Kolkata`, `America/New_York` |

Language codes are Tesseract's three-letter codes (`eng`, `deu`, `fra`, `spa`,
`ita`, `hin`, `tam`, ...). Changing `OCR_LANGUAGES` requires a rebuild (step 3).

## 3. Build and start

```bash
docker compose up -d --build
```

The first build takes 10–20 minutes (it compiles the UI and installs the OCR
toolchain). The first start then takes another minute or two while migrations
run. Watch progress with:

```bash
docker compose ps
```

```bash
docker compose logs -f web
```

When `web` shows `healthy`, open **http://SERVER_IP:12000** and sign in with
`ADMIN_USERNAME` / `ADMIN_PASSWORD`.

Upload a PDF, JPEG, PNG or TIFF; the `worker` container OCRs it in the
background (`docker compose logs -f worker`) and its text becomes searchable.

### Using the prebuilt upstream image instead

To skip the local build, set in `.env`:

```
PAPERMERGE_IMAGE=papermerge/papermerge:3.0.2
```

then run `docker compose pull web worker && docker compose up -d`. Note that
the upstream image only includes German (`deu`) and English OCR data, and does
not contain changes made in this repository.

## 4. Enable HTTPS (recommended for anything internet-facing)

1. Point your domain's DNS at the server.
2. In `.env` set `DOMAIN=dms.yourdomain.com` and `HTTP_BIND=127.0.0.1`
   (so the unencrypted port is reachable only through the proxy).
3. Start with the `https` profile:

```bash
docker compose --profile https up -d
```

Caddy obtains and renews a Let's Encrypt certificate automatically. Visit
**https://dms.yourdomain.com**.

If you already run a reverse proxy (nginx, Traefik, Cloudflare Tunnel, ...),
skip the profile and proxy to `http://127.0.0.1:12000`. Remember to forward
WebSocket upgrades for `/ws/` and allow request bodies of at least 100 MB.

## Operations

All commands run from the `deploy/` directory.

| Task | Command |
|------|---------|
| Status | `docker compose ps` |
| Logs | `docker compose logs -f web worker` |
| Stop | `docker compose down` (data is kept in volumes) |
| Restart | `docker compose restart` |
| More OCR throughput | `docker compose up -d --scale worker=3` |
| List users | `docker compose exec web list_users.sh` |
| Create API token | `docker compose exec web create_token.sh <username>` |
| Django shell / manage.py | `docker compose exec web bash -c "cd /core_app && poetry run ./manage.py shell"` |

### Upgrading

```bash
git pull
```

```bash
docker compose up -d --build
```

Migrations run automatically when `web` starts. Back up first (below).

### Backups

Your data is in three Docker volumes: `papermerge_postgres_data`,
`papermerge_media` (original files and OCR output) and `papermerge_index_db`
(search index, can be rebuilt). Back up the first two.

Database dump:

```bash
docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > papermerge-db-$(date +%F).sql
```

Document files:

```bash
docker run --rm -v papermerge_media:/data -v "$PWD":/backup alpine tar czf /backup/papermerge-media-$(date +%F).tar.gz -C /data .
```

Restore (on a fresh deployment, with the stack stopped except `db`):

```bash
docker compose up -d db
```

```bash
docker compose exec -T db sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"' < papermerge-db-YYYY-MM-DD.sql
```

```bash
docker run --rm -v papermerge_media:/data -v "$PWD":/backup alpine tar xzf /backup/papermerge-media-YYYY-MM-DD.tar.gz -C /data
```

```bash
docker compose up -d
```

Papermerge also has its own archive format: `docker compose exec web backup.sh /core_app/media/backup.tar`
and `restore.sh`.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `required variable ... is missing a value` | Fill that variable in `deploy/.env` |
| `web` stays `starting` / `unhealthy` | `docker compose logs web`; usually a wrong `DB_PASSWORD` after the DB volume was already created with another one. Use the original password, or wipe with `docker compose down -v` (**deletes all data**) |
| Documents upload but text is never searchable | Check `docker compose logs worker`; make sure the worker is running and the document's OCR language is installed (`OCR_LANGUAGES`) |
| Upload of large files fails | Default limit is 100 MB per request (nginx in `docker/prod/nginx.conf` and the `Caddyfile`) |
| Login loops behind your own proxy | Proxy must pass the `Host` header and cookies unchanged, and serve the app at the domain root (`/`) |
| Changed `ADMIN_PASSWORD` has no effect | The admin is only created on the first start; change the password in the UI (Users) instead |

## Local development

For development with live reload see `dev.yml` and the "Tests" section of the
[README](./README.md). The deployment above is for running Papermerge, not for
editing it.
