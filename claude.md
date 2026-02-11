# Claude.md — Infrastructure Repo Context

This is the **infrastructure-only** repository for [amare.wedding](https://amare.wedding). It contains Docker Compose orchestration, Nginx reverse proxy config, and a GitHub Actions deployment pipeline. No application code lives here.

## What this repo manages

Two domains on a single Scaleway VPS (fr-par region):

- **amare.wedding** — Wedding planning SaaS (API + Frontend)
- **l-craft.be** — Personal portfolio site

## Services (docker-compose.yml)

| Service | Container | Image | Port |
|---------|-----------|-------|------|
| API | `amare-api` | `rg.fr-par.scw.cloud/amare-wedding/api:latest` | 8080 |
| Frontend | `amare-frontend` | `rg.fr-par.scw.cloud/amare-wedding/frontend:latest` | 80 |
| Portfolio | `portfolio` | `rg.fr-par.scw.cloud/amare-wedding/portfolio:latest` | 80 |
| Nginx | `amare-nginx` | `nginx:alpine` | 80, 443 |

All on bridge network `amare-network`. All have health checks (30s interval, 3 retries) and log rotation (10MB, 3 files).

API receives Scaleway env vars: `SCW_SECRET_KEY`, `SCW_DEFAULT_ORGANIZATION_ID`, `SCW_DEFAULT_PROJECT_ID`, `SCW_REGION`.

## Nginx routing (nginx/conf.d/default.conf)

Uses upstream blocks with DNS resolver (`127.0.0.11`) for Docker service discovery.

**amare.wedding:**
- `/api/*` → `api:8080` (rate limit: 100 req/min, burst 10)
- `/health` → `api:8080/health` (no rate limit)
- `/*` → `frontend:80` (rate limit: 1000 req/min, burst 50)

**l-craft.be:**
- `/*` → `portfolio:80` (rate limit: 1000 req/min, burst 50)

Both domains: HTTP→HTTPS redirect, Let's Encrypt SSL, hidden dotfiles.

## Deployment (.github/workflows/deploy-infra.yml)

Triggers on push to `main` (when `docker-compose.yml`, `nginx/**`, or the workflow changes) or manual dispatch.

Steps:
1. Dynamically fetches VPS IP from Scaleway API (no hardcoded IP)
2. SSHs to VPS as root
3. Backs up current config to `/app/backups/` with timestamp
4. Clones fresh config from GitHub
5. Validates docker-compose + nginx syntax
6. Pulls latest images, runs `docker-compose up -d`
7. Rolls back from backup if validation fails

### Required GitHub Secrets

`VPS_SSH_KEY`, `SCW_SECRET_KEY`, `SCW_DEFAULT_ORGANIZATION_ID`, `SCW_DEFAULT_PROJECT_ID`, `SCW_ZONE`, `SCW_INSTANCE_ID`

## Key paths on VPS

- `/app/` — Docker Compose + Nginx config
- `/app/.env` — Scaleway secrets (written by deploy workflow)
- `/app/backups/` — Timestamped backups
- `/etc/letsencrypt/` — SSL certificates

## When making changes

- Always validate syntax (`docker-compose config`, `nginx -t`) before pushing
- Pushes to `main` deploy automatically — use feature branches for testing
- Ports and paths are depended on by the API and frontend — don't change without coordinating
- Rate limits, SSL config, and secrets management are security features — preserve them

## Related repositories

- **API:** `Sillves/WeddingManagerApi` (.NET 10, PostgreSQL)
- **Frontend:** `Sillves/amare-wedding-frontend` (React 19, TypeScript)
- **Infrastructure:** This repo (`Sillves/amare-wedding-infrastructure`)

## What this repo does NOT contain

- Application source code
- Database schemas or migrations
- Business logic
- Frontend components
