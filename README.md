# Amare.Wedding Infrastructure

Infrastructure-as-Code for the Amare.Wedding (Vowly) platform. Manage Docker, Nginx, and deployment configuration via GitHub.

## Overview

This repository contains all infrastructure configuration for Amare.Wedding deployed on Scaleway VPS:
- Docker Compose setup
- Nginx reverse proxy configuration
- Rate limiting rules
- Automated deployments via GitHub Actions

**Live:** https://amare.wedding

## Repository Structure

```
amare-wedding-infrastructure/
├── docker-compose.yml          # Main Docker Compose configuration
├── nginx/
│   ├── nginx.conf              # Main nginx configuration
│   └── conf.d/
│       └── default.conf        # Server blocks & rate limiting
├── .github/workflows/
│   └── deploy-infra.yml        # Infrastructure deployment workflow
├── backups/                    # Automatic backups on deploy
└── README.md
```

## Quick Start

### Local Setup

1. **Clone repository**
```bash
git clone https://github.com/Sillves/amare-wedding-infrastructure.git
cd amare-wedding-infrastructure
```

2. **Test locally**
```bash
docker-compose up -d

# Verify services
docker-compose ps

# Check health
curl http://localhost/health
curl http://localhost/api/health
```

3. **View logs**
```bash
docker-compose logs -f nginx
docker-compose logs -f api
docker-compose logs -f frontend
```

### Making Changes

1. **Edit configuration** (docker-compose.yml or nginx files)
2. **Test locally**
3. **Commit and push to main**
4. **GitHub Actions automatically deploys to VPS**

## Docker Compose Services

### API Service
- Image: `rg.fr-par.scw.cloud/amare-wedding/api:latest`
- Port: 8080 (internal only, via nginx proxy)
- Health Check: GET /health every 30s
- Restart: unless-stopped
- Environment: Production .NET configuration

### Frontend Service
- Image: `rg.fr-par.scw.cloud/amare-wedding/frontend:latest`
- Port: 80 (internal only, via nginx proxy)
- Health Check: HTTP GET / every 30s
- Restart: unless-stopped

### Nginx Service
- Image: nginx:alpine
- Ports: 80 (HTTP), 443 (HTTPS)
- Role: Reverse proxy + rate limiting
- Config: /etc/nginx/conf.d/default.conf

### Network
- Type: Bridge network named `amare-network`
- All services communicate internally via network

## Nginx Configuration

### Rate Limiting

**API Endpoints** (`/api/`):
- Limit: 100 requests per minute per IP
- Burst: 10 extra requests allowed
- Status: 429 Too Many Requests on violation

**Frontend** (`/`):
- Limit: 1000 requests per minute per IP
- Burst: 50 extra requests allowed
- Status: 429 Too Many Requests on violation

**Health Check** (`/health`):
- No rate limiting (for monitoring)

### Proxy Headers

All requests include:
- `X-Real-IP`: Client IP
- `X-Forwarded-For`: Proxy chain
- `X-Forwarded-Proto`: Original protocol (http/https)
- `Host`: Original host header

### SSL/TLS

- Certificates: `/etc/letsencrypt/live/amare.wedding/`
- Protocol: TLSv1.2 + TLSv1.3
- Ciphers: HIGH security level

## Deployment

### Automatic Deployment

When you push changes to `main` branch:

1. GitHub Actions triggered
2. Code checked out
3. Infrastructure pulled to VPS
4. Docker Compose syntax validated
5. Nginx syntax validated
6. Services reloaded
7. Health checks verified
8. Backup created

### Deployment Path

```
git push origin main
  ↓
GitHub Actions: deploy-infra.yml
  ↓
SSH to VPS
  ↓
Clone latest config
  ↓
Validate syntax
  ↓
Reload services
  ↓
Verify health
  ↓
✅ Live!
```

### Automatic Rollback

If deployment fails:
1. Nginx syntax validation fails → rollback to previous config
2. Docker Compose validation fails → rollback to previous config
3. Backup created before each deployment

Backups are stored in `/app/backups/` on VPS with timestamp.

## Secrets & Environment Variables

Required GitHub Secrets:
- `VPS_IP`: VPS public IP address
- `VPS_SSH_KEY`: Private SSH key for VPS access

Environment variables injected by API/Frontend CI/CD:
- `SCW_SECRET_KEY`
- `SCW_DEFAULT_ORGANIZATION_ID`
- `SCW_DEFAULT_PROJECT_ID`
- `SCW_REGION`

## Monitoring

### Container Status
```bash
docker-compose ps
```

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f nginx
docker-compose logs -f api
docker-compose logs -f frontend

# Last 100 lines
docker-compose logs --tail=100 nginx
```

### Health Checks
```bash
# Local
curl http://localhost/health
curl http://localhost/api/health

# Production
curl https://amare.wedding/health
curl https://amare.wedding/api/health
```

### Disk Usage
```bash
df -h
du -sh /app
```

### Log Rotation

Logs are automatically rotated:
- Max file size: 10MB
- Max files: 3
- Driver: json-file
- Old logs automatically deleted

## Troubleshooting

### Nginx won't reload
```bash
# Validate syntax
docker exec amare-nginx nginx -t

# View error
docker logs amare-nginx | tail -20

# Manual reload
docker exec amare-nginx nginx -s reload
```

### API/Frontend not responding
```bash
# Check container status
docker-compose ps

# View logs
docker-compose logs api
docker-compose logs frontend

# Restart service
docker-compose restart api
docker-compose restart frontend
```

### Deployment failed
1. Check GitHub Actions logs
2. SSH to VPS: `ssh root@<VPS_IP>`
3. Check backups: `ls -la /app/backups/`
4. Restore if needed: `cp /app/backups/docker-compose.yml.* /app/docker-compose.yml`

### High disk usage
```bash
# Check sizes
du -sh /app/*

# Clean old backups
rm /app/backups/docker-compose.yml.202601*

# Clean Docker
docker system prune -a
```

## Security

### Best Practices

✅ **Implemented:**
- Secrets via GitHub Secrets (not in code)
- SSH key-based deployment
- Rate limiting on endpoints
- No sensitive data in logs
- Automatic backups before changes
- Syntax validation before deploy
- Health checks on all services

### SSH Key Management

Private key stored as GitHub Secret: `VPS_SSH_KEY`
- 4096-bit RSA
- Only used in GitHub Actions
- Rotate every 6 months

## Scaling

### Multiple API Instances

Update docker-compose.yml:
```yaml
api:
  deploy:
    replicas: 3
```

Load balance with nginx:
```nginx
upstream api_backend {
    server api:8080;
    server api_2:8080;
    server api_3:8080;
}
```

### Resource Limits

Add to docker-compose.yml:
```yaml
api:
  deploy:
    resources:
      limits:
        cpus: '1'
        memory: 512M
```

## Backup & Restore

### Automatic Backups

Backups created before each deployment:
```bash
/app/backups/
├── docker-compose.yml.20260124_090000
├── docker-compose.yml.20260124_091500
└── nginx.20260124_090000/
```

### Manual Backup

```bash
# On VPS
cd /app
cp docker-compose.yml backups/docker-compose.yml.backup
cp -r nginx backups/nginx.backup
```

### Restore

```bash
# On VPS
cd /app
cp backups/docker-compose.yml.20260124_090000 ./docker-compose.yml
docker-compose down
docker-compose up -d
```

## Contributing

1. Create feature branch: `git checkout -b fix/rate-limit-config`
2. Make changes
3. Test locally: `docker-compose up`
4. Commit: `git commit -m "fix: adjust rate limits"`
5. Push: `git push origin fix/rate-limit-config`
6. Create Pull Request

**Branch naming:**
- `fix/*` - Configuration fixes
- `feature/*` - New services/features
- `chore/*` - Maintenance tasks

## Related Repositories

- **API:** https://github.com/Sillves/WeddingManagerApi
- **Frontend:** https://github.com/Sillves/amare-wedding-frontend

## Support

Issues or questions:
1. Check deployment logs in GitHub Actions
2. SSH to VPS and inspect logs
3. Check backups and restore if needed
4. Create GitHub issue in this repo

---

**Last Updated:** January 24, 2026  
**Current Version:** 1.0.0  
**Status:** Production Ready ✅
