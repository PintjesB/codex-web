# codex-web

A browser frontend for Codex Desktop, deployed as a local-only Docker stack on a machine you control.

https://github.com/user-attachments/assets/0a33cbd8-741c-412c-9e75-46dfe9324596

## Purpose

`codex-web` brings the Codex Desktop experience to a browser while keeping the backend on your own dev server. The intended deployment for this fork is Docker Compose, bound to localhost only.

```text
Docker Compose
├── codex-app-server   -> Unix socket in Docker volume
├── codex-web          -> 127.0.0.1:8214
└── opencode-web       -> 127.0.0.1:4096
```

No public reverse proxy is required. Access from another machine should use SSH local forwarding or a VPN.

## Features in this fork

- Dockerized `codex-web` image.
- Dockerized `codex-app-server` image.
- Local-only Docker Compose deployment.
- Optional OpenCode Web sidecar.
- Optional GHCR image publishing workflow.
- GHCR pull-based Compose deployment.
- Workspace root scoping via `CODEX_WEB_WORKSPACE_ROOT`.
- Upload size limit via `CODEX_WEB_MAX_UPLOAD_BYTES`.
- Guarded `/@fs/` access.
- Non-loopback bind protection unless explicitly allowed.

## Repository layout

```text
.
├── Dockerfile.codex-app-server
├── Dockerfile.codex-web
├── deploy/container
│   ├── .env.example
│   ├── README.md
│   ├── compose.ghcr.yml
│   └── compose.yml
└── .github/workflows
    ├── ci.yml
    └── publish-containers.yml
```

## Option A: build locally

Create folders on the dev server:

```bash
sudo mkdir -p /srv/dev/apps /srv/dev/workspace /srv/dev/compose/codex-web
sudo chown -R "$USER:$USER" /srv/dev
```

Clone and configure:

```bash
cd /srv/dev/apps
git clone git@github.com:PintjesB/codex-web.git

cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.yml ./compose.yml
cp /srv/dev/apps/codex-web/deploy/container/.env.example ./.env
nano .env
```

Start:

```bash
cd /srv/dev/compose/codex-web
docker compose build
docker compose up -d
```

## Option B: pull GHCR images

After the `Publish containers` workflow has published images from `main`, use the GHCR Compose file.

Published images:

```text
ghcr.io/pintjesb/codex-web:latest
ghcr.io/pintjesb/codex-app-server:latest
```

Configure:

```bash
cd /srv/dev/apps
git clone git@github.com:PintjesB/codex-web.git

cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.ghcr.yml ./compose.yml
cp /srv/dev/apps/codex-web/deploy/container/.env.example ./.env
nano .env
```

Set:

```env
GHCR_OWNER=pintjesb
IMAGE_TAG=latest
```

Pull and start:

```bash
cd /srv/dev/compose/codex-web
docker compose pull
docker compose up -d
```

If the GHCR packages are private, run `docker login ghcr.io` first.

## Authenticate Codex

Codex authentication is stored in the persistent `codex-home` Docker volume.

```bash
cd /srv/dev/compose/codex-web
docker compose run --rm codex-app-server codex login --device-auth
docker compose restart codex-app-server codex-web
```

## Access

Use SSH local forwarding from your desktop:

```bash
ssh -N \
  -L 8214:127.0.0.1:8214 \
  -L 4096:127.0.0.1:4096 \
  dev@dev-ai-01
```

Then open:

```text
http://127.0.0.1:8214
http://127.0.0.1:4096
```

Verify local-only binding on the dev server:

```bash
ss -ltnp | grep -E ':(8214|4096)'
```

Expected bind address: `127.0.0.1`, not `0.0.0.0`.

## Update local-build deployment

```bash
cd /srv/dev/apps/codex-web
git pull

cd /srv/dev/compose/codex-web
docker compose build --pull
docker compose up -d
```

## Update GHCR deployment

```bash
cd /srv/dev/apps/codex-web
git pull

cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.ghcr.yml ./compose.yml
docker compose pull
docker compose up -d
```

## CI/CD

| Workflow | Purpose |
|---|---|
| `CI` | Installs dependencies without lifecycle scripts, builds the server, and runs advisory dependency audit. |
| `Publish containers` | Builds images on pull requests and publishes GHCR images on `main`, `v*` tags, or manual dispatch. |

Pull requests build images without pushing. Pushes to `main` publish `latest` and SHA tags.

## Configuration

| Variable | Default | Purpose |
|---|---:|---|
| `DEV_WORKSPACE` | `/srv/dev/workspace` | Host workspace mounted into containers as `/workspace`. |
| `CODEX_WEB_PORT` | `8214` | Localhost port for Codex Web. |
| `CODEX_WEB_MAX_UPLOAD_BYTES` | `26214400` | Multipart upload size limit. |
| `CODEX_BUFFER_SIZE` | `104857600` | WebSocket proxy buffer size. |
| `OPENCODE_WEB_PORT` | `4096` | Localhost port for OpenCode Web. |
| `OPENCODE_SERVER_PASSWORD` | required | Password for OpenCode Web. |
| `GHCR_OWNER` | `pintjesb` | GHCR owner used by `compose.ghcr.yml`. |
| `IMAGE_TAG` | `latest` | GHCR image tag used by `compose.ghcr.yml`. |

## Security model

The web UI provides powerful access to the mounted workspace. Keep the services bound to localhost, access them through SSH forwarding or VPN, and mount only the workspace directories you intend the agents to use.

More details are in:

```text
deploy/container/README.md
```
