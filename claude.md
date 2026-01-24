# Claude.md - AI Assistant Context

This document provides context for AI assistants (like Claude) working on the Amare Wedding infrastructure repository.

## Repository Purpose

This is the **infrastructure-as-code** repository for the Amare Wedding (Vowly) platform. It manages:

- Docker Compose orchestration for microservices
- Nginx reverse proxy configuration and rate limiting
- Automated deployment pipelines via GitHub Actions
- Production environment configuration for Scaleway VPS

**Do not** expect application code here - this repository only contains infrastructure configuration.

## Project Context

### What is Amare Wedding?
A wedding management platform (also known as Vowly) that helps couples plan and manage their weddings. The platform consists of:

1. **API Service**: ASP.NET Core backend (separate repository)
2. **Frontend Service**: Web application (separate repository)
3. **Infrastructure**: This repository - manages deployment configuration

### Related Repositories
- API: `Sillves/WeddingManagerApi`
- Frontend: `Sillves/amare-wedding-frontend`
- Infrastructure: `Sillves/amare-wedding-infrastructure` (this repo)

## Architecture Overview

```
Internet → Nginx (Port 80/443)
            ├─→ Frontend Service (Internal Port 80)
            └─→ API Service (Internal Port 8080)
```

### Services

1. **Nginx** (`nginx:alpine`)
   - Acts as reverse proxy
   - Handles SSL/TLS termination
   - Implements rate limiting
   - Serves on ports 80 (HTTP) and 443 (HTTPS)

2. **API** (`rg.fr-par.scw.cloud/amare-wedding/api:latest`)
   - ASP.NET Core application
   - Internal port 8080
   - Health check endpoint: `/health`
   - Uses Scaleway secrets for configuration

3. **Frontend** (`rg.fr-par.scw.cloud/amare-wedding/frontend:latest`)
   - Web application
   - Internal port 80
   - Health check on root endpoint

### Network
All services communicate via a bridge network named `amare-network`.

## Key Files

### `docker-compose.yml`
Main orchestration file defining all services, networks, health checks, and logging configuration.

**Important details:**
- Log rotation: 10MB max, 3 files
- Health checks run every 30s with 3 retries
- Restart policy: `unless-stopped`
- Environment variables for Scaleway credentials

### `nginx/nginx.conf`
Main Nginx configuration with:
- Worker process settings (auto-scaling)
- Gzip compression for web assets
- 20MB client body size limit
- TCP optimization (sendfile, tcp_nopush, tcp_nodelay)

### `nginx/conf.d/default.conf`
Server block configuration including:
- Rate limiting rules (100 req/min for API, 1000 req/min for frontend)
- Proxy settings for API and frontend
- SSL/TLS configuration
- Health check endpoint routing

### `.github/workflows/deploy-infra.yml`
Automated deployment pipeline that:
1. Backs up current configuration
2. Clones latest config from GitHub
3. Validates Docker Compose syntax
4. Validates Nginx configuration
5. Pulls latest container images
6. Restarts services
7. Verifies deployment health

## Deployment Flow

### Automatic Deployment
Triggered on:
- Push to `main` branch
- Changes to `docker-compose.yml`, `nginx/**`, or workflow files
- Manual workflow dispatch

### Deployment Steps
1. Checkout code from GitHub
2. SSH to VPS using stored credentials
3. Create timestamped backups
4. Clone fresh infrastructure config
5. Copy new files to `/app` directory
6. Validate configurations
7. Pull latest container images
8. Restart services with `docker-compose up -d`
9. Verify deployment status

### Rollback Strategy
If validation fails:
- Nginx syntax error → restore from backup
- Docker Compose syntax error → restore from backup
- Backups stored in `/app/backups/` with timestamps

## Environment & Secrets

### GitHub Secrets (Required)
- `VPS_IP`: Production server IP address
- `VPS_SSH_KEY`: SSH private key for deployment
- `SCW_SECRET_KEY`: Scaleway API secret key
- `SCW_DEFAULT_ORGANIZATION_ID`: Scaleway organization ID
- `SCW_DEFAULT_PROJECT_ID`: Scaleway project ID

### Environment Variables
- `ASPNETCORE_ENVIRONMENT=Production`
- `ASPNETCORE_URLS=http://+:8080`
- `SCW_REGION=fr-par` (default)

## Common Tasks

### Testing Configuration Changes Locally
```bash
# Validate docker-compose syntax
docker-compose config

# Start services locally
docker-compose up -d

# Check health
curl http://localhost/health
curl http://localhost/api/health
```

### Updating Nginx Configuration
1. Edit files in `nginx/` directory
2. Test locally with `docker exec amare-nginx nginx -t`
3. Commit and push to `main`
4. GitHub Actions will deploy automatically

### Checking Deployment Status
```bash
# On VPS
docker-compose ps
docker-compose logs -f

# Check specific service
docker-compose logs api --tail=50
```

### Manual Deployment to VPS
```bash
# SSH to VPS
ssh root@<VPS_IP>

# Navigate to app directory
cd /app

# Pull latest images
docker-compose pull

# Restart services
docker-compose up -d
```

## Rate Limiting Rules

### API Endpoints (`/api/*`)
- **Limit**: 100 requests per minute per IP
- **Burst**: 10 additional requests allowed
- **Response**: 429 Too Many Requests when exceeded

### Frontend (`/*`)
- **Limit**: 1000 requests per minute per IP
- **Burst**: 50 additional requests allowed
- **Response**: 429 Too Many Requests when exceeded

### Health Check (`/health`)
- **No rate limiting** - allows monitoring services unrestricted access

## Security Considerations

### Implemented Security Features
- SSL/TLS encryption via Let's Encrypt
- Rate limiting on all public endpoints
- Secrets management via GitHub Secrets (not in code)
- SSH key-based deployment
- Read-only Nginx configuration volumes
- Automatic configuration validation before deployment
- No sensitive data in logs

### Security Best Practices
- Never commit secrets or credentials
- Rotate SSH keys every 6 months
- Review rate limiting rules regularly
- Monitor logs for suspicious activity
- Keep backups of working configurations

## Monitoring & Logging

### Log Locations
- **Nginx**: `/var/log/nginx/access.log` and `/var/log/nginx/error.log`
- **Docker**: JSON file driver with automatic rotation

### Health Checks
All services have health checks configured:
- Interval: 30 seconds
- Timeout: 10 seconds
- Retries: 3
- Start period: 40 seconds

### Viewing Logs
```bash
# All services
docker-compose logs -f

# Specific service with tail
docker-compose logs --tail=100 nginx

# Follow API logs
docker-compose logs -f api
```

## Troubleshooting

### Common Issues

**Nginx won't start**
- Check syntax: `docker exec amare-nginx nginx -t`
- Review logs: `docker logs amare-nginx`
- Verify SSL certificates exist in `/etc/letsencrypt`

**Services not responding**
- Check status: `docker-compose ps`
- View logs: `docker-compose logs <service>`
- Restart: `docker-compose restart <service>`

**Deployment failed**
- Check GitHub Actions logs
- SSH to VPS and inspect `/app` directory
- Review recent backups in `/app/backups/`
- Restore from backup if needed

**High disk usage**
- Clean Docker: `docker system prune -a`
- Remove old backups: `rm /app/backups/*`
- Check log sizes: `du -sh /var/lib/docker/`

## Development Guidelines

### Making Infrastructure Changes

1. **Test Locally First**
   - Always run `docker-compose config` to validate syntax
   - Test changes with `docker-compose up -d`
   - Verify health checks pass

2. **Use Descriptive Commits**
   - `fix:` for configuration fixes
   - `feat:` for new services/features
   - `chore:` for maintenance tasks

3. **Branch Strategy**
   - `main` branch deploys to production automatically
   - Use feature branches for testing: `fix/nginx-rate-limit`
   - Create PR for review before merging

4. **Validation Checklist**
   - [ ] Docker Compose syntax valid
   - [ ] Nginx configuration valid
   - [ ] Health checks configured
   - [ ] Secrets not in code
   - [ ] Logging configured
   - [ ] Tested locally

## Important Notes for AI Assistants

### What This Repository Does NOT Contain
- Application source code (API or Frontend)
- Database schemas or migrations
- Business logic
- Frontend components

### What This Repository DOES Contain
- Docker Compose orchestration
- Nginx reverse proxy configuration
- Deployment automation
- Infrastructure configuration only

### When Making Changes
- **Always validate syntax** before suggesting changes
- **Consider deployment impact** - changes deploy automatically on push to main
- **Maintain backward compatibility** - services depend on specific ports/paths
- **Preserve security features** - rate limiting, SSL, secrets management
- **Update documentation** if changing architecture or deployment process

### Testing Philosophy
- Test locally before production deployment
- Use the automatic rollback feature by validating configurations
- Monitor deployment logs in GitHub Actions
- Keep backups before making changes

### Scaling Considerations
- Current setup is single-instance per service
- To scale API, use Docker Compose `replicas` and update Nginx upstream config
- Database is external (not in this infrastructure)
- Static assets served through Nginx

## Quick Reference

### Useful Commands
```bash
# Validate configurations
docker-compose config
docker exec amare-nginx nginx -t

# View status
docker-compose ps

# Restart services
docker-compose restart

# View logs
docker-compose logs -f

# Pull latest images
docker-compose pull

# Deploy changes
docker-compose up -d

# Clean up
docker system prune -a
```

### Important Paths
- **VPS App Directory**: `/app`
- **Backups**: `/app/backups/`
- **SSL Certificates**: `/etc/letsencrypt`
- **Nginx Logs**: `/var/log/nginx/`

### Key URLs
- **Production**: https://amare.wedding
- **Health Check**: https://amare.wedding/health
- **API Health**: https://amare.wedding/api/health

## Version Information

- **Current Version**: 1.0.0
- **Last Updated**: January 24, 2026
- **Status**: Production Ready ✅
- **Deployment**: Scaleway VPS (fr-par region)

---

**Note**: This document is intended to provide context for AI assistants. For user-facing documentation, see [README.md](README.md).
