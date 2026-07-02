# Local-only container deployment

This is the preferred deployment model when the dev server should keep all agent services inside Docker and expose them only on the dev server loopback interface.

It runs:

- `codex-app-server` as a container with Codex CLI installed.
- `codex-web` as a container built from this repository.
- `opencode-web` from the official OpenCode image.

No reverse proxy is required. Ports are bound to `127.0.0.1` only.

## Network exposure

The Compose file publishes only loopback ports:

```text
127.0.0.1:8214 -> codex-web
127.0.0.1:4096 -> opencode-web
```

To access the UIs from your desktop, use SSH local forwarding:

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

## Folder layout on the dev server

```text
/srv/dev
├── apps
│   └── codex-web
├── workspace
│   ├── repo-a
│   └── repo-b
└── compose
    └── codex-web
```

Create it:

```bash
sudo mkdir -p /srv/dev/apps /srv/dev/workspace /srv/dev/compose/codex-web
sudo chown -R "$USER:$USER" /srv/dev
```

## Clone and configure

```bash
cd /srv/dev/apps
git clone git@github.com:PintjesB/codex-web.git
cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.yml ./compose.yml
cp /srv/dev/apps/codex-web/deploy/container/.env.example ./.env
nano .env
```

Set at minimum:

```env
DEV_WORKSPACE=/srv/dev/workspace
OPENCODE_SERVER_PASSWORD=replace-with-a-long-random-password
OPENROUTER_API_KEY=replace-if-used
OPENAI_API_KEY=replace-if-used
ANTHROPIC_API_KEY=replace-if-used
```

## Build and start

```bash
cd /srv/dev/compose/codex-web
docker compose build
docker compose up -d
```

## Authenticate Codex

Codex auth is stored in the `codex-home` Docker volume. Run the login flow inside a one-off container:

```bash
cd /srv/dev/compose/codex-web
docker compose run --rm codex-app-server codex login --device-auth
docker compose restart codex-app-server codex-web
```

## Check status

```bash
docker compose ps
docker compose logs -f codex-app-server
docker compose logs -f codex-web
docker compose logs -f opencode-web
```

## Verify local-only access

On the dev server:

```bash
curl -I http://127.0.0.1:8214
curl -I http://127.0.0.1:4096
ss -ltnp | grep -E ':(8214|4096)'
```

Expected bind address: `127.0.0.1`, not `0.0.0.0`.

## Update

```bash
cd /srv/dev/apps/codex-web
git pull
cd /srv/dev/compose/codex-web
docker compose build --pull
docker compose up -d
```

## Do we need CI/CD image publishing?

No, not for this local-only deployment.

The dev server can build the images directly from the checked-out repository. A private GHCR image can be added later when the container setup is proven stable, but it is not required for the first deployment.
