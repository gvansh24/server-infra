# Server-infra — Code Review

Reviewed: 2026-07-20. Scope: `nginx.conf`, `routes/*.conf`, `docker-compose.yml`,
`.github/workflows/deploy.yml`. This repo is the master nginx gateway that fronts all
games on one host via a shared external Docker network `web-gateway`.

## Findings

### 1. CI deploy targets the DEAD server — every `deploy.yml` (Critical)
All three repos' `.github/workflows/deploy.yml` do:
```yaml
host: ${{ secrets.SERVER_HOST }}
script: |
  cd ~/chess-app          # wrong path on the new box
  git pull origin master
  docker compose up -d --build   # no sudo; ubuntu isn't in the docker group
```
Three problems after the migration to openclaw (`92.4.88.247`):
- **`secrets.SERVER_HOST`** still points at the terminated server IP.
- **Path** is `~/chess-app`; on openclaw the repos live under `~/server/<repo>`.
- **`docker` needs sudo** on openclaw (the `ubuntu` user isn't in the `docker` group),
  and the volume-mount of `web-gateway` must already exist.

**Consequence:** a push to `master` fires the Action and it fails against a dead host.
- **Fix (all three repos):** update the workflow `cd` path to `~/server/<repo>` and use
  `sudo docker compose ...`; and in GitHub → repo → Settings → Secrets update
  `SERVER_HOST` (→ `92.4.88.247`), `SERVER_USER` (`ubuntu`), and `SSH_PRIVATE_KEY`
  (the `personal` key). Alternatively add `ubuntu` to the `docker` group on the box
  (`sudo usermod -aG docker ubuntu`) so `sudo` isn't needed — cleaner for CI.
  Until secrets are fixed, either disable the workflow or expect red builds.

### 2. No TLS — `nginx.conf` (Medium)
The gateway only `listen 80;`. Port 443 is open on the box/OCI but nothing serves it,
so games are HTTP-only. The chess WS correctly upgrades to `wss:` when the page is
HTTPS (`game.js`), so it's ready for TLS the moment you add it.
- **Fix:** add a certbot/Let's Encrypt sidecar or terminate TLS at the gateway; you'll
  need a domain first (currently IP-only).

### 3. Undocumented required manual step — `docker-compose.yml` (Low)
Every compose file references `web-gateway` as an `external: true` network, which must
be created by hand (`docker network create web-gateway`) before first `up`. This isn't
captured anywhere and will trip a fresh deploy.
- **Fix:** add a one-line bootstrap note to the README, or a `make bootstrap` /
  `deploy.sh` that creates the network idempotently before bringing services up.

### 4. Single gateway, no health checks (Informational)
One nginx container fronts everything with `restart: always` — fine for a hobby host.
Consider Docker `healthcheck:` blocks on the game containers so a wedged backend is
visibly unhealthy rather than silently 502-ing through the proxy.

## What's already good
- Clean modular routing: `include /etc/nginx/routes/*.conf;` so each game is one drop-in
  conf — genuinely nice, adding a game is a single file.
- Correct Docker DNS usage (`resolver 127.0.0.11`, variable `proxy_pass`) so nginx
  starts even if an upstream is briefly down instead of failing config load.
- Proper WebSocket upgrade headers on the chess route; long read/send timeouts now set
  for idle game sockets.
- Separating infra from app repos is the right call — the gateway evolves independently.
