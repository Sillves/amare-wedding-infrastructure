# Amare Wedding Infrastructure

Infrastructure-as-code for [amare.wedding](https://amare.wedding) and [l-craft.be](https://l-craft.be), deployed on a Scaleway VPS.

## What's in here

- `docker-compose.yml` — 4 services: API, Frontend, Portfolio, Nginx
- `nginx/` — Reverse proxy config, rate limiting, SSL termination
- `.github/workflows/deploy-infra.yml` — Auto-deploy on push to main

No application code lives here. See the [related repos](#related-repositories) for that.

## Architecture

```
Internet
  │
  ├─ amare.wedding ──→ Nginx (80/443)
  │                      ├─→ Frontend (internal :80)
  │                      └─→ API (internal :8080)
  │
  └─ l-craft.be ─────→ Nginx (80/443)
                         └─→ Portfolio (internal :80)
```

All services run on a single VPS and communicate via a Docker bridge network (`amare-network`).

## Services

| Service | Image | Internal Port | Health Check |
|---------|-------|---------------|-------------|
| API | `rg.fr-par.scw.cloud/amare-wedding/api:latest` | 8080 | `GET /health` |
| Frontend | `rg.fr-par.scw.cloud/amare-wedding/frontend:latest` | 80 | `GET /` |
| Portfolio | `rg.fr-par.scw.cloud/amare-wedding/portfolio:latest` | 80 | `GET /` |
| Nginx | `nginx:alpine` | 80, 443 | — |

All services: `restart: unless-stopped`, health checks every 30s (3 retries), log rotation at 10MB (3 files).

## Rate Limiting

| Route | Limit | Burst |
|-------|-------|-------|
| `/api/*` (amare.wedding) | 100 req/min per IP | 10 |
| `/*` (amare.wedding) | 1000 req/min per IP | 50 |
| `/*` (l-craft.be) | 1000 req/min per IP | 50 |
| `/health` | No limit | — |

Exceeding the limit returns `429 Too Many Requests`.

## Deployment

Pushes to `main` (affecting `docker-compose.yml`, `nginx/**`, or the workflow) trigger automatic deployment via GitHub Actions:

1. Fetch VPS public IP from Scaleway API
2. SSH to VPS
3. Backup current config (`/app/backups/`)
4. Clone latest infra from GitHub
5. Validate docker-compose and nginx syntax
6. Pull latest container images
7. `docker-compose up -d`
8. Verify deployment

If validation fails, the backup is restored automatically.

### Required GitHub Secrets

| Secret | Purpose |
|--------|---------|
| `VPS_SSH_KEY` | SSH private key for VPS |
| `SCW_SECRET_KEY` | Scaleway API key |
| `SCW_DEFAULT_ORGANIZATION_ID` | Scaleway org ID |
| `SCW_DEFAULT_PROJECT_ID` | Scaleway project ID |
| `SCW_ZONE` | Scaleway zone (e.g. `fr-par-1`) |
| `SCW_INSTANCE_ID` | Scaleway VPS instance ID |

## Local Testing

```bash
docker-compose config          # Validate syntax
docker-compose up -d           # Start services
docker-compose ps              # Check status
docker-compose logs -f nginx   # Follow logs
```

## Troubleshooting

```bash
# Nginx won't start
docker exec amare-nginx nginx -t
docker logs amare-nginx

# Service not responding
docker-compose ps
docker-compose logs <service>
docker-compose restart <service>

# Disk full
docker system prune -a
rm /app/backups/*
```

## Repository Structure

```
amare-wedding-infrastructure/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── default.conf
├── .github/workflows/
│   └── deploy-infra.yml
├── claude.md
└── README.md
```

## Related Repositories

- **API:** [Sillves/WeddingManagerApi](https://github.com/Sillves/WeddingManagerApi)
- **Frontend:** [Sillves/amare-wedding-frontend](https://github.com/Sillves/amare-wedding-frontend)

---

**Last Updated:** February 11, 2026
