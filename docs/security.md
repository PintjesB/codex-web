# Runtime security

`codex-web` bridges a browser UI to Codex running as the server-side OS user. Treat access to the web UI as equivalent to SSH access for that user.

## Mandatory deployment controls

- Bind `codex-web` to `127.0.0.1` by default.
- Put Traefik, Authelia, VPN, or another trusted authentication boundary in front of it.
- Set `CODEX_WEB_WORKSPACE_ROOT` to the only folder tree that users may browse through the UI.
- Do not run the process as `root`.
- Do not mount or expose the Docker socket to the service.
- Use a dedicated Linux user for Codex and `codex-web`.

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `CODEX_WEB_WORKSPACE_ROOT` | current user's home directory | Root directory allowed for workspace browsing and `/@fs/` file serving. Production deployments should set this to `/srv/dev/workspace` or another dedicated repo root. |
| `CODEX_WEB_MAX_UPLOAD_BYTES` | `26214400` | Maximum accepted size for an uploaded file. |
| `CODEX_WEB_ALLOW_NON_LOOPBACK` | unset | Must be set to `1`, `true`, `yes`, or `on` before `--host` may bind to a non-loopback interface. |

## File serving boundary

The browser client uses `/@fs/` URLs for local file display. The server only serves files whose real path is inside one of these roots:

1. `CODEX_WEB_WORKSPACE_ROOT`
2. the per-process temporary upload directory

The server uses `realpath` checks before serving files so symlinks inside the workspace cannot be used to read host files outside the allowed roots.

## Recommended production settings

```bash
CODEX_WEB_WORKSPACE_ROOT=/srv/dev/workspace
CODEX_WEB_MAX_UPLOAD_BYTES=26214400
```

Keep the process listening on localhost:

```bash
node src/server/main.js --host 127.0.0.1 --port 8214
```

If you deliberately bind to all interfaces, make that decision explicit:

```bash
CODEX_WEB_ALLOW_NON_LOOPBACK=1 node src/server/main.js --host 0.0.0.0 --port 8214
```

Do not use the non-loopback mode without a strong external authentication layer.
