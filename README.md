# **Self-Hosted Immich Deployment on Hetzner Cloud**

## Introduction

This repository contains the production-grade Docker Compose configuration for my personal self-hosted Immich deployment—a privacy-focused, open-source alternative to Google Photos. Immich provides comprehensive photo and video backup, organization, and AI-powered features while maintaining complete data ownership and control over personal media assets.

## Architecture Overview

This deployment orchestrates a multi-container architecture using Docker Compose:

- **Immich Server**: Core application server handling API requests, media ingestion, and user management
- **Immich Machine Learning**: Dedicated service for AI-powered features including face recognition, object detection, and smart search using vector embeddings
- **PostgreSQL Database**: Specialized PostgreSQL 14 instance with `vectorchord` and `pgvectors` extensions for advanced vector search capabilities
- **Valkey (Redis)**: High-performance caching layer for session management and temporary data storage

All services are configured with persistent storage, health checks, automatic restart policies, and proper service dependencies to ensure reliable operation.

## Prerequisites

- Docker Engine 20.10+ and Docker Compose v2.0+
- Hetzner Cloud VPS (or equivalent cloud provider) with:
  - Minimum 2 CPU cores, 4GB RAM (recommended: 4+ cores, 8GB+ RAM for ML workloads)
  - Sufficient storage capacity for your media library (plan for growth)
  - Ubuntu 22.04 LTS or Debian 12 (recommended)
- Domain name with DNS A record pointing to your VPS IP (for HTTPS)
- Basic familiarity with Docker, Linux command line, and reverse proxy configuration

## Installation

### 1. Clone Repository

```bash
git clone https://github.com/YOUR_USERNAME/selfhosted-immichapp-cloud.git
cd selfhosted-immichapp-cloud
```

### 2. Configure Environment Variables

Copy the example environment file and configure your settings:

```bash
cp .env.example .env
nano .env  # or use your preferred editor
```

**Critical variables to configure:**

- `IMMICH_VERSION`: Immich release version (default: `release` for latest stable)
- `UPLOAD_LOCATION`: Absolute host path for media library storage (e.g., `/mnt/storage/immich/library`)
- `DB_DATA_LOCATION`: Absolute host path for PostgreSQL data directory (e.g., `/mnt/storage/immich/postgres`)
- `DB_PASSWORD`: Strong PostgreSQL password (generate with `openssl rand -base64 32`)
- `DB_USERNAME`: PostgreSQL username (default: `postgres`)
- `DB_DATABASE_NAME`: Database name (default: `immich`)

### 3. Create Storage Directories

Ensure storage directories exist with proper permissions:

```bash
sudo mkdir -p ${UPLOAD_LOCATION} ${DB_DATA_LOCATION}
sudo chown -R 1000:1000 ${UPLOAD_LOCATION}  # Immich runs as UID 1000
sudo chown -R 999:999 ${DB_DATA_LOCATION}   # PostgreSQL runs as UID 999
```

### 4. Deploy Services

Start all services in detached mode:

```bash
docker-compose up -d
```

### 5. Verify Deployment

Check service status:

```bash
docker-compose ps
```

All services should show `Up` status. View logs if needed:

```bash
docker-compose logs -f immich-server
```

## Configuration

### Environment Variables

The `.env` file centralizes all configuration. Key variables include:

- **Application**: `IMMICH_VERSION`, `UPLOAD_LOCATION`
- **Database**: `DB_PASSWORD`, `DB_USERNAME`, `DB_DATABASE_NAME`, `DB_DATA_LOCATION`
- **Network**: Port mappings (default: `2283` for Immich server)
- **Security**: All sensitive credentials should be stored in `.env` (never commit to git)

### Reverse Proxy Setup (Nginx)

Immich should be accessed through a reverse proxy for HTTPS and security. Example Nginx configuration:

```nginx
server {
    listen 443 ssl http2;
    server_name photos.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/photos.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/photos.yourdomain.com/privkey.pem;

    # Trust proxy headers
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;

    # WebSocket support
    location / {
        proxy_pass http://localhost:2283;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

Obtain SSL certificate using Certbot:

```bash
sudo certbot --nginx -d photos.yourdomain.com
```

### Hardware Acceleration (Optional)

For improved transcoding and ML inference performance, uncomment and configure hardware acceleration in `docker-compose.yml`:

- **Transcoding**: NVENC (NVIDIA), QuickSync (Intel), VAAPI (AMD/Intel)
- **ML Inference**: CUDA (NVIDIA), ROCm (AMD), OpenVINO (Intel)

Refer to [Immich hardware acceleration documentation](https://immich.app/docs/guides/hardware-transcoding) for detailed setup.

## Data Persistence

### Volume Mounts

- **Media Library**: `${UPLOAD_LOCATION}` → `/usr/src/app/upload` (all user-uploaded photos and videos)
- **PostgreSQL Data**: `${DB_DATA_LOCATION}` → `/var/lib/postgresql/data` (database files, indexes, WAL)
- **ML Model Cache**: `model-cache` (Docker named volume) → `/cache` (downloaded ML models)

### Backup Strategy

**Critical**: Implement regular backups for both media library and database.

**Database Backup** (recommended: daily):

```bash
docker-compose exec database pg_dump -U ${DB_USERNAME} ${DB_DATABASE_NAME} > backup_$(date +%Y%m%d).sql
```

**Media Library Backup** (recommended: weekly or as needed):

```bash
rsync -avz ${UPLOAD_LOCATION}/ /backup/location/immich-library/
```

Consider automating backups with cron jobs or dedicated backup solutions (BorgBackup, Restic, etc.).

## Maintenance

### Updating Immich

1. Pull latest images:
   ```bash
   docker-compose pull
   ```

2. Restart services:
   ```bash
   docker-compose up -d
   ```

3. Verify health:
   ```bash
   docker-compose ps
   docker-compose logs --tail=50 immich-server
   ```

### Monitoring Logs

View logs for all services:
```bash
docker-compose logs -f
```

View logs for specific service:
```bash
docker-compose logs -f immich-server
docker-compose logs -f immich-machine-learning
```

### Database Maintenance

PostgreSQL is configured with data checksums enabled for integrity verification. Monitor database size and perform periodic VACUUM operations if needed:

```bash
docker-compose exec database psql -U ${DB_USERNAME} -d ${DB_DATABASE_NAME} -c "VACUUM ANALYZE;"
```

## Security Considerations

- **Environment Variables**: Never commit `.env` file to version control. Use `.env.example` as a template.
- **Reverse Proxy**: Always use HTTPS. Expose only ports 80/443 on firewall, not port 2283 directly.
- **Firewall Rules**: Configure UFW or firewalld to restrict access:
  ```bash
  sudo ufw allow 22/tcp    # SSH
  sudo ufw allow 80/tcp     # HTTP (for Let's Encrypt)
  sudo ufw allow 443/tcp    # HTTPS
  sudo ufw enable
  ```
- **Updates**: Regularly update Docker images, host OS, and dependencies.
- **Credentials**: Use strong, unique passwords. Consider password managers for credential storage.

## Performance Optimization

- **Storage Type**: PostgreSQL performance varies significantly between HDD and SSD. For production workloads, prefer SSD storage for database volumes.
- **Resource Allocation**: Monitor resource usage (`docker stats`) and adjust container limits if needed in `docker-compose.yml`.
- **Hardware Acceleration**: Enable GPU acceleration for transcoding and ML inference if supported hardware is available.
- **Database Tuning**: Adjust PostgreSQL shared_buffers and other parameters based on available RAM (requires custom postgresql.conf).

## Troubleshooting

### Container Startup Failures

- Check logs: `docker-compose logs [service-name]`
- Verify environment variables: `docker-compose config`
- Ensure storage directories exist with correct permissions
- Verify port 2283 is not already in use: `sudo netstat -tulpn | grep 2283`

### Database Connection Errors

- Verify database service is running: `docker-compose ps database`
- Check database logs: `docker-compose logs database`
- Confirm `.env` variables match: `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE_NAME`
- Test connection: `docker-compose exec database psql -U ${DB_USERNAME} -d ${DB_DATABASE_NAME}`

### Storage Permission Issues

- Verify ownership: `ls -la ${UPLOAD_LOCATION}`
- Fix permissions: `sudo chown -R 1000:1000 ${UPLOAD_LOCATION}`
- Check SELinux/AppArmor if applicable

### Reverse Proxy Issues

- Verify Immich is accessible locally: `curl http://localhost:2283/api/server-info/ping`
- Check Nginx error logs: `sudo tail -f /var/log/nginx/error.log`
- Ensure WebSocket headers are configured correctly
- Verify trusted proxy settings in Immich configuration

## References

- [Immich Official Documentation](https://immich.app/docs)
- [Immich Docker Compose Installation Guide](https://immich.app/docs/install/docker-compose)
- [Immich GitHub Releases](https://github.com/immich-app/immich/releases)
- [Immich Hardware Acceleration Guide](https://immich.app/docs/guides/hardware-transcoding)
- [Immich Community Discord](https://discord.gg/immich)

## License

This repository contains deployment configuration files. Immich itself is licensed under the [MIT License](https://github.com/immich-app/immich/blob/main/LICENSE).

---

**Note**: This is a personal deployment configuration. Adapt paths, domain names, and settings to your specific infrastructure requirements.
