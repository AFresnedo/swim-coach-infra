# SwimCoach Infra

[![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)](LICENSE)

Docker Compose stack for the production deployment of [SwimCoach](https://swim-coach-ai.com),
a website for personalized, AI-assisted swim instruction. This repo owns the shared
`docker-compose.yml` and Caddy config that the back-end and front-end CD pipelines deploy
into — it has no application code of its own. Clone this repo to `/infra` on the Droplet.

Sibling repos: [back-end](https://github.com/AFresnedo/swim-coach-back-end) ·
[front-end](https://github.com/AFresnedo/swim-coach-front-end)

## Architecture

Five services on one Droplet, networked together via the `swimcoach` Docker network
(see [`docker-compose.yml`](docker-compose.yml)):

- **caddy** — reverse proxy and automatic TLS for `${DOMAIN}`; redirects `www` to the
  naked domain and sets security headers (HSTS, `X-Frame-Options`, etc. — see
  [`Caddyfile`](Caddyfile))
- **frontend** — the Next.js app, pinned to a fixed internal IP so the backend can trust
  its `X-Forwarded-For` header as the real client IP
- **backend** — the FastAPI app; waits on `postgres` and `redis` to report healthy before
  starting
- **postgres** — `pgvector/pgvector:pg17`, with the `vector` extension enabling the
  back-end's RAG knowledge base
- **redis** — password-protected; backs the back-end's rate limiter and the training
  endpoint's feature-flag toggle

## Environment variables

All variables are documented in [`.env.example`](.env.example); the notable ones:

| Variable | Notes |
| --- | --- |
| `DOMAIN` | the public domain Caddy requests a TLS cert for |
| `BACKEND_IMAGE` / `FRONTEND_IMAGE` | GHCR image refs; the CD pipelines update these on deploy |
| `POSTGRES_PASSWORD` / `REDIS_PASSWORD` | generate strong random values, e.g. `openssl rand -hex 32` |
| `DATABASE_URL` | backend's Postgres connection string, built from `POSTGRES_PASSWORD` |
| `SECRET_KEY` | backend's JWT signing key, e.g. `openssl rand -hex 32` |
| `API_URL` | frontend → backend URL inside the Docker network (`http://backend:8000`) |
| `SITE_INDEXABLE` | prod-only launch gate; keep `false` until the site should be crawlable |
| `TURNSTILE_SECRET_KEY` | Cloudflare Turnstile secret; frontend fails closed (rejects all sign-ups) if unset |

## One-time Droplet setup

These steps are not automated and must be repeated if the Droplet is ever rebuilt.

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
```

### 2. Create deploy user
```bash
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
mkdir -p /home/deploy/.ssh
cat <swimcoach_deploy.pub contents> >> /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

### 3. Clone this repo
```bash
git clone https://github.com/AFresnedo/swim-coach-infra.git /infra
chown -R deploy:deploy /infra
```

### 4. Create .env
Copy `.env.example` to `.env` and fill in all values (see Environment variables above).

### 5. Start postgres
```bash
docker compose -f /infra/docker-compose.yml up -d postgres
```

### 6. Enable pgvector
```bash
docker compose -f /infra/docker-compose.yml exec postgres psql -U postgres -d swimcoach -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### 7. Start redis
```bash
docker compose -f /infra/docker-compose.yml up -d redis
```
The CD pipeline's deploy step runs `docker compose up -d --no-deps backend`, which never
starts `redis` itself - it must already be running and healthy or `backend` will never
satisfy its `depends_on: condition: service_healthy` and fail to start.

### 8. Enable unattended security upgrades with auto-reboot
Edit `/etc/apt/apt.conf.d/50unattended-upgrades` and uncomment:
```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

### 9. Start Caddy
```bash
docker compose -f /infra/docker-compose.yml up -d --no-deps caddy
```

Once this setup is complete, the backend and frontend are deployed automatically by
their own repos' CD pipelines — see the next section.

## Day-2 operations

**Deploys** happen automatically: a push to `main` in the back-end or front-end repo
builds a Docker image, pushes it to GHCR, then SSHes in and runs
`docker compose up -d --no-deps <service>` (backend also runs Alembic migrations first),
finishing with a smoke test against `/health`. Nothing here needs to be run by hand for
a normal deploy.

**Manual redeploy / rollback** — pull a specific tag (every CD build is tagged with its
commit SHA, in addition to `latest`) and recreate just that service:
```bash
# in /infra/.env, set BACKEND_IMAGE or FRONTEND_IMAGE to the sha tag you want
docker compose up -d --no-deps backend   # or frontend
```

**Logs**:
```bash
docker compose logs -f backend   # or frontend / postgres / redis / caddy
```

**Status**:
```bash
docker compose ps
```

## Docs

- [`docs/auth-smoke-test.md`](docs/auth-smoke-test.md) — manual curl-based smoke test of
  the register → logout → sign-in flow through the frontend's BFF proxy

## License

All rights reserved — see [LICENSE](LICENSE). This repo is public for reference and
portfolio purposes only; no permission is granted to use, copy, modify, or distribute
this code, and it isn't open to outside contributions. Matches the
[back-end](https://github.com/AFresnedo/swim-coach-back-end/blob/main/LICENSE) and
[front-end](https://github.com/AFresnedo/swim-coach-front-end/blob/main/LICENSE) licenses.
