# Dify Dokploy Deployment Guide

This is a minimal Docker Compose configuration for deploying Dify on Dokploy with PostgreSQL and Weaviate.

## Services Included

### Core Services

- **API**: Main backend API service
- **Worker**: Background task processor (Celery worker)
- **Worker Beat**: Task scheduler (Celery beat)
- **Web**: Frontend web application (Next.js)

### Infrastructure Services

- **PostgreSQL**: Primary database
- **Redis**: Cache and message queue
- **Weaviate**: Vector database for embeddings
- **Sandbox**: Code execution environment
- **SSRF Proxy**: Security proxy for outbound requests (simplified for Dokploy)
- **Nginx**: Reverse proxy and single entry point for Traefik routing

## Quick Start

### 1. Copy Environment Variables

```bash
cp .env.dokploy .env
```

### 2. Configure Your Environment

Edit the `.env` file and update:

#### Required Changes:

- **SECRET_KEY**: Generate a strong key:
  ```bash
  openssl rand -base64 42
  ```

#### Recommended Changes:

- **Database Password**: Update `DB_PASSWORD`
- **Redis Password**: Update `REDIS_PASSWORD`
- **Weaviate API Key**: Update `WEAVIATE_API_KEY`
- **Sandbox API Key**: Update `SANDBOX_API_KEY`

#### Dokploy URLs:

Update these based on your Dokploy domain:

```env
CONSOLE_WEB_URL=https://dify.yourdomain.com
CONSOLE_API_URL=https://dify.yourdomain.com
SERVICE_API_URL=https://dify.yourdomain.com
APP_WEB_URL=https://dify.yourdomain.com
APP_API_URL=https://dify.yourdomain.com
FILES_URL=https://dify.yourdomain.com
```

### 3. Deploy to Dokploy

#### Option A: Using Dokploy UI

1. Go to your Dokploy dashboard
2. Create a new service → Docker Compose
3. Upload `docker-compose.dokploy.yaml`
4. Add environment variables from `.env.dokploy`
5. Deploy

#### Option B: Using CLI

```bash
cd docker
docker compose -f docker-compose.dokploy.yaml up -d
```

## Volume Management

The configuration uses named volumes for data persistence:

- `db_data`: PostgreSQL database files
- `redis_data`: Redis persistence
- `weaviate_data`: Weaviate vector database
- `app_storage`: User uploaded files and assets
- `sandbox_dependencies`: Sandbox Python dependencies
- `sandbox_conf`: Sandbox configuration

## Port Mapping

For Dokploy deployment with Traefik, you only need to expose **ONE PORT**:

### Single Port Exposure:

**Nginx Service**

- **Internal Port**: `80`
- **Purpose**: Single entry point that routes to both web (frontend) and api (backend)
- **Dokploy Config**: Map this port in Dokploy/Traefik

### How it Works:

```
Traefik (Dokploy) → nginx:80 → {
                                    web:3000 (frontend)
                                    api:5001 (backend API)
                                  }
```

### Route Mapping:

- `/` → Web frontend
- `/api/*` → Backend API
- `/console/api/*` → Console API
- `/v1/*` → API v1 endpoints
- `/files/*` → File serving
- `/explore` → Web explore page

### Dokploy Configuration:

1. In Dokploy, expose **port 80** from the nginx service
2. Point your domain to this service
3. Traefik will handle SSL/TLS automatically
4. All routing is handled internally by nginx

**Note**: The api and web services do NOT need external port mappings - nginx handles all internal routing.

## Health Checks

All services include health checks:

- **PostgreSQL**: Uses `pg_isready`
- **Redis**: Uses `redis-cli ping`
- **Weaviate**: HTTP endpoint check
- **Sandbox**: HTTP health endpoint

Services will wait for dependencies to be healthy before starting.

## Scaling

You can scale the worker services for better performance:

```bash
docker compose -f docker-compose.dokploy.yaml up -d --scale worker=3
```

Or configure scaling in Dokploy UI.

## Troubleshooting

### Check Service Logs

```bash
docker compose -f docker-compose.dokploy.yaml logs -f [service_name]
```

### Common Issues

1. **Getting 404 Errors**

   - **Check URL Configuration**: Ensure all URL variables are set in `.env`
   - **For Dokploy**: Update URLs to your actual domain:
     ```env
     CONSOLE_WEB_URL=https://dify.yourdomain.com
     CONSOLE_API_URL=https://dify.yourdomain.com
     SERVICE_API_URL=https://dify.yourdomain.com
     APP_WEB_URL=https://dify.yourdomain.com
     APP_API_URL=https://dify.yourdomain.com
     FILES_URL=https://dify.yourdomain.com
     ```
   - **For Local Testing**: Use localhost URLs:
     ```env
     CONSOLE_WEB_URL=http://localhost
     CONSOLE_API_URL=http://localhost:5001
     SERVICE_API_URL=http://localhost:5001
     APP_WEB_URL=http://localhost
     APP_API_URL=http://localhost:5001
     FILES_URL=http://localhost:5001
     ```
   - **Verify Services**: Check that API and Web are running: `docker compose ps`
   - **Check Logs**: `docker compose logs web api`

2. **Database Connection Failed**

   - Ensure PostgreSQL is healthy: `docker compose ps`
   - Check database credentials in `.env`

3. **Redis Connection Failed**

   - Verify Redis password matches in all services
   - Check Redis is running: `docker compose ps redis`

4. **Weaviate Connection Failed**

   - Ensure Weaviate API key is correctly set
   - Verify Weaviate endpoint: `http://weaviate:8080`

5. **File Upload Issues**
   - Ensure `FILES_URL` is set to your public domain
   - Check `app_storage` volume permissions

### Reset Everything

```bash
docker compose -f docker-compose.dokploy.yaml down -v
docker compose -f docker-compose.dokploy.yaml up -d
```

⚠️ **Warning**: This will delete all data including database and uploads.

## Security Recommendations

1. **Change All Default Passwords**: Update all passwords in `.env`
2. **Use Strong SECRET_KEY**: Generate with `openssl rand -base64 42`
3. **Enable HTTPS**: Configure SSL in Dokploy (handled by Dokploy)
4. **Regular Backups**: Backup the volumes, especially `db_data` and `app_storage`
5. **Update Images**: Regularly update to latest stable versions

## Backup

### Database Backup

```bash
docker compose -f docker-compose.dokploy.yaml exec db pg_dump -U postgres dify > backup.sql
```

### Restore Database

```bash
cat backup.sql | docker compose -f docker-compose.dokploy.yaml exec -T db psql -U postgres dify
```

### Volume Backup

```bash
docker run --rm -v docker_db_data:/data -v $(pwd):/backup alpine tar czf /backup/db_data.tar.gz /data
```

## Monitoring

Monitor your deployment:

```bash
# Check all services
docker compose -f docker-compose.dokploy.yaml ps

# Check resource usage
docker stats

# View logs
docker compose -f docker-compose.dokploy.yaml logs -f
```

## Updating

To update to a new version:

1. Update image tags in `docker-compose.dokploy.yaml`
2. Pull new images:
   ```bash
   docker compose -f docker-compose.dokploy.yaml pull
   ```
3. Restart services:
   ```bash
   docker compose -f docker-compose.dokploy.yaml up -d
   ```

## Support

For more information:

- [Dify Documentation](https://docs.dify.ai)
- [Dokploy Documentation](https://dokploy.com/docs)
- [GitHub Issues](https://github.com/langgenius/dify/issues)
