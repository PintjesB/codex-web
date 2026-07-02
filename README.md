# codex-web

A browser frontend for Codex Desktop, running on a machine you control.

https://github.com/user-attachments/assets/0a33cbd8-741c-412c-9e75-46dfe9324596

## Motivation

The agents were never meant to stay trapped in a terminal window for long.
Codex Desktop brought the power of agents to your local computer, where your
files, credentials, and tools already live.

`codex-web` brings Codex Desktop to the browser while keeping the backend on a
machine you control: a Linux dev server, homelab VM, cloud VM, desktop, or Mac
mini. Agents can keep running after your laptop closes, and you can reconnect
from any device with a browser.

This project aims to be as thin a wrapper as possible so upstream changes to the
Codex Desktop app can be integrated quickly.

## What this fork adds

This fork is prepared for an always-on, local-only dev-server deployment:

- Dockerized `codex-web` image.
- Dockerized `codex-app-server` image.
- Local-only Docker Compose deployment bound to `127.0.0.1`.
- Optional GHCR image publishing workflow.
- GHCR pull-based Compose deployment.
- Workspace root scoping via `CODEX_WEB_WORKSPACE_ROOT`.
- Upload size limit via `CODEX_WEB_MAX_UPLOAD_BYTES`.
- Guarded `/@fs/` access so host-root file serving is not exposed globally.
- Non-loopback bind protection unless explicitly allowed.
- Dedicated deployment docs under `deploy/container/` and `deploy/`.

## Recommended deployment: local-only containers

Use this when the service should run on a dev server and be reachable only from
that server itself or through an SSH tunnel.

```text
Docker Compose
├── codex-app-server   -> Unix socket in Docker volume
├── codex-web          -> 127.0.0.1:8214
└── opencode-web       -> 127.0.0.1:4096
```

Create the server folders:

```bash
sudo mkdir -p /srv/dev/apps /srv/dev/workspace /srv/dev/compose/codex-web
sudo chown -R "$USER:$USER" /srv/dev
```

Clone the repository and prepare the Compose directory:

```bash
cd /srv/dev/apps
git clone git@github.com:PintjesB/codex-web.git

cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.yml ./compose.yml
cp /srv/dev/apps/codex-web/deploy/container/.env.example ./.env
nano .env
```

Build and start locally:

```bash
cd /srv/dev/compose/codex-web
docker compose build
docker compose up -d
```

Authenticate Codex inside the persistent `codex-home` Docker volume:

```bash
cd /srv/dev/compose/codex-web
docker compose run --rm codex-app-server codex login --device-auth
docker compose restart codex-app-server codex-web
```

Access from your desktop through SSH local forwarding:

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

Expected: `127.0.0.1`, not `0.0.0.0`.

## GHCR deployment

The `Publish containers` workflow builds images on pull requests and publishes
them to GHCR on pushes to `main`, version tags, or manual dispatch.

Published images:

```text
ghcr.io/pintjesb/codex-web:latest
ghcr.io/pintjesb/codex-app-server:latest
```

After the workflow has published images, switch the dev server to the GHCR
Compose file:

```bash
cd /srv/dev/apps/codex-web
git pull

cd /srv/dev/compose/codex-web
cp /srv/dev/apps/codex-web/deploy/container/compose.ghcr.yml ./compose.yml
nano .env
docker compose pull
docker compose up -d
```

Set these values in `.env` when using GHCR images:

```env
GHCR_OWNER=pintjesb
IMAGE_TAG=latest
```

If the packages are private, authenticate Docker on the dev server first:

```bash
docker login ghcr.io
```

## Host usage

`codex-web` can still be run directly on a host. By default, it listens on
`127.0.0.1:8214`.

It uses `codex` from `PATH` if available, or `CODEX_CLI_PATH` if set.

Run with `npx`:

```bash
npx --yes github:0xcaff/codex-web
```

Or with Nix:

```bash
nix run github:0xcaff/codex-web
```

Then open:

```text
http://127.0.0.1:8214
```

Ensure the Codex CLI on the host machine is signed in before starting the
server:

```bash
codex login --device-auth
```

## App-server proxy mode

It is often useful to run the app server separately so a crash or restart of
`codex-web` does not interrupt the Codex process executing commands.

Start a long-lived app server:

```bash
codex app-server --listen unix:///tmp/codex-app-server.sock
```

Then run `codex-web` with the proxy helper:

```bash
nix shell github:0xcaff/codex-web github:0xcaff/codex-web#codex_remote_proxy -c bash -lc '
  export CODEX_UNIX_SOCKET=/tmp/codex-app-server.sock
  export CODEX_CLI_PATH="$(command -v codex_remote_proxy)"
  codex-web
'
```

The container deployment uses this same pattern internally: `codex-app-server`
exposes a Unix socket in a Docker volume, and `codex-web` connects to it through
`codex_remote_proxy`.

## Configuration

| Variable | Default | Purpose |
|---|---:|---|
| `CODEX_WEB_WORKSPACE_ROOT` | user home, container uses `/workspace` | Restricts workspace browsing and `/@fs/` file serving to an allowed root. |
| `CODEX_WEB_MAX_UPLOAD_BYTES` | `26214400` | Multipart upload size limit. |
| `CODEX_WEB_ALLOW_NON_LOOPBACK` | unset | Must be set to allow binding to non-loopback addresses. |
| `CODEX_CLI_PATH` | `codex` from `PATH` | Path to Codex CLI or `codex_remote_proxy`. |
| `CODEX_UNIX_SOCKET` | unset | Unix socket used by `codex_remote_proxy`. |
| `CODEX_BUFFER_SIZE` | `104857600` | WebSocket proxy buffer size. |

## Security

Run `codex-web` only on trusted networks. Treat anyone who can reach the web UI
as someone who can operate Codex as the user running the service.

Someone with access to the web UI may be able to:

- Run commands with the permissions of the `codex-web`/Codex process.
- Read or modify files, environment variables, credentials, SSH keys, and other
  resources available to that process.
- Use the signed-in Codex / ChatGPT account and consume usage quota or billing
  credits.

Recommended exposure model:

- Bind browser services to `127.0.0.1` only.
- Access remotely with SSH local forwarding or a VPN.
- Use a reverse proxy and authentication gateway only if you intentionally want
  browser access beyond localhost.

## Features

- Browser frontend for Codex Desktop.
- Localhost-first operation.
- Docker Compose deployment for dev servers.
- Optional GHCR image publishing and pull-based deployment.
- Long-lived app-server proxy mode.
- Workspace-root-aware file access.
- Upload limits.
- Working today:
  - subagents
  - inline images
  - editor sidepanel
  - transcription

## Roadmap

Some parts of the desktop experience are not wired up yet:

- browser panel support, likely rebuilt around iframes
- computer use on linux, which could become a very powerful feature
- terminal support
- git worker integration
- whatever else people find and file issues for

## Issues welcome

If something is broken, missing, or rough around the edges, please file an
issue.

Using `codex-web` in an interesting way? Post about it on X and tag
[@0xcaff](https://x.com/0xcaff).

Using this at a company and need something more tailored? Email me and we can
talk.

## Alternatives

- [davej/pocodex](https://github.com/davej/pocodex) was useful until this
  project needed subagents and an inline image viewer.
- The native Codex remote feature is useful for connecting to remote Codex hosts
  over SSH to manage long-running tasks, but it requires Codex Desktop on the
  client device.
- Upcoming first-party mobile app from OpenAI. `codex-web` exists and works
  today.
