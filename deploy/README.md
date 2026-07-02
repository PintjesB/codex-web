# Dedicated dev server deployment

This guide deploys an always-on development VM with:

- Codex app-server on a Unix socket.
- Codex Web bound to localhost.
- OpenCode Web bound to localhost.
- optional code-server in Docker.
- Traefik/Authelia in front of all browser access.

The browser UIs are remote shells/agents. Treat access to them as equivalent to SSH access for the `dev` user.

## Target layout

```text
/srv/dev
├── apps
│   └── codex-web
├── workspace
│   ├── repo-a
│   └── repo-b
├── secrets
│   ├── codex-web.env
│   └── opencode.env
├── code-server
│   └── config
└── compose
    └── code-server
```

## 1. VM baseline

Recommended VM:

| Resource | Recommendation |
|---|---:|
| OS | Ubuntu Server 24.04 |
| vCPU | 4 to 8 |
| RAM | 16 to 32 GB |
| Disk | 150 to 250 GB SSD |
| Network | internal VLAN or VPN only |

Install base packages:

```bash
sudo apt update
sudo apt install -y \
  curl \
  git \
  jq \
  unzip \
  patch \
  build-essential \
  python3 \
  python3-pip \
  python3-venv \
  tmux \
  ripgrep \
  fd-find \
  ca-certificates \
  gnupg \
  openssh-client \
  websocat
```

Install Docker:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
```

Install Node.js 22:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node --version
npm --version
```

## 2. Create the dev user

```bash
sudo adduser dev
sudo usermod -aG docker dev
sudo loginctl enable-linger dev

sudo mkdir -p \
  /srv/dev/apps \
  /srv/dev/workspace \
  /srv/dev/secrets \
  /srv/dev/logs \
  /srv/dev/compose

sudo chown -R dev:dev /srv/dev
sudo chmod 700 /srv/dev/secrets
```

## 3. Install Codex and OpenCode

```bash
sudo -iu dev
npm config set prefix ~/.local
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.profile
export PATH="$HOME/.local/bin:$PATH"

npm install -g @openai/codex opencode-ai

which codex
which opencode
```

Authenticate Codex:

```bash
codex login --device-auth
```

## 4. Clone and build Codex Web

```bash
sudo -iu dev
cd /srv/dev/apps
git clone git@github.com:PintjesB/codex-web.git
cd /srv/dev/apps/codex-web
npm ci
npm run build
chmod +x /srv/dev/apps/codex-web/scripts/codex_remote_proxy
```

## 5. Create env files

```bash
sudo -iu dev
cp /srv/dev/apps/codex-web/deploy/env/codex-web.env.example /srv/dev/secrets/codex-web.env
cp /srv/dev/apps/codex-web/deploy/env/opencode.env.example /srv/dev/secrets/opencode.env
chmod 600 /srv/dev/secrets/*.env
```

Edit the secrets:

```bash
nano /srv/dev/secrets/codex-web.env
nano /srv/dev/secrets/opencode.env
```

## 6. Install user services

```bash
sudo -iu dev
mkdir -p ~/.config/systemd/user
cp /srv/dev/apps/codex-web/deploy/systemd/*.service ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable --now codex-app-server.service
systemctl --user enable --now codex-web.service
systemctl --user enable --now opencode-web.service
```

Check status:

```bash
systemctl --user status codex-app-server.service
systemctl --user status codex-web.service
systemctl --user status opencode-web.service
```

View logs:

```bash
journalctl --user -u codex-app-server.service -f
journalctl --user -u codex-web.service -f
journalctl --user -u opencode-web.service -f
```

## 7. Local verification

```bash
curl -I http://127.0.0.1:8214
curl -I http://127.0.0.1:4096
```

After the runtime hardening PR is merged, verify host file escape is blocked:

```bash
curl -i http://127.0.0.1:8214/@fs/etc/passwd
```

Expected: no `/etc/passwd` contents are returned.

## 8. Optional code-server

```bash
sudo -iu dev
mkdir -p /srv/dev/compose/code-server
cp /srv/dev/apps/codex-web/deploy/compose/code-server.compose.yml /srv/dev/compose/code-server/compose.yml
cd /srv/dev/compose/code-server
nano compose.yml
```

Set the correct `PUID` and `PGID`:

```bash
id dev
```

Start code-server:

```bash
docker compose up -d
```

## 9. Traefik

Use `deploy/traefik/dynamic.yml.example` as a starting point. Keep the backend services bound to localhost and terminate TLS/auth at Traefik.

Example hostnames:

```text
codex-dev.example.tld     -> 127.0.0.1:8214
opencode-dev.example.tld  -> 127.0.0.1:4096
code-dev.example.tld      -> 127.0.0.1:8443
```

Use Authelia or VPN in front of every route.

## 10. Update procedure

```bash
sudo -iu dev
cd /srv/dev/apps/codex-web
git pull
npm ci
npm run build
systemctl --user restart codex-web.service
```

Restart backends:

```bash
systemctl --user restart codex-app-server.service
systemctl --user restart codex-web.service
systemctl --user restart opencode-web.service
```

## 11. Repo workflow policy

Put an `AGENTS.md` in each repo:

```markdown
# Agent rules

- Always work on a feature branch.
- Never push directly to main or master.
- Use branch names under `ai/`.
- Run relevant tests before committing.
- Summarize changed files before asking to push.
- Do not modify secrets, production config, SSH keys, or deployment credentials.
- Do not use the Docker host socket unless explicitly instructed.
- Prefer small commits with clear messages.
```

Protect `main` or `master` on GitHub and allow AI pushes only to task branches.
