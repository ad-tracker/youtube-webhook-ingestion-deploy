# YouTube Webhook Ingestion - Deployment

Production-ready Docker Compose deployment for the [YouTube Webhook Ingestion Service](https://github.com/ad-tracker/youtube-webhook-ingestion-go) with automatic SSL/TLS certificates via Swag reverse proxy.

## Features

- **Complete Stack**: PostgreSQL database, webhook service, and Swag reverse proxy
- **Automatic SSL/TLS**: Let's Encrypt certificates with auto-renewal
- **Security**: Rate limiting, secure headers, and API key authentication
- **High Availability**: Health checks and automatic container restarts
- **Easy Configuration**: Simple `.env` file for all settings
- **Production Ready**: Optimized for production deployments

## Architecture

```
Internet
   |
   v
Swag (Port 80/443)
   ├─ SSL/TLS Termination
   ├─ Rate Limiting
   └─ Reverse Proxy
       |
       v
YouTube Webhook Service (Port 8080)
   ├─ /webhook - YouTube notifications
   ├─ /api/v1/subscriptions - Subscription management
   └─ /health - Health check
       |
       v
PostgreSQL Database
   └─ Persistent storage
```

## Prerequisites

- **Server**: Linux server with a public IP address
- **Domain**: A domain name pointed to your server's IP address
- **Docker**: Docker Engine 20.10+ and Docker Compose V2
- **Ports**: Ports 80 and 443 must be open and accessible from the internet

### Install Docker

If you don't have Docker installed:

```bash
# Install Docker (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/ad-tracker/youtube-webhook-ingestion-deploy.git
cd youtube-webhook-ingestion-deploy
```

### 2. Configure Environment Variables

Copy the example environment file and edit it with your settings:

```bash
cp .env.example .env
chmod 600 .env  # Restrict permissions for security
nano .env       # or use your preferred editor
```

**Required Configuration**:

1. **DOMAIN**: Your domain name (e.g., `webhooks.example.com`)
2. **POSTGRES_PASSWORD**: Generate a strong password:
   ```bash
   openssl rand -base64 32
   ```
3. **API_KEYS**: Generate one or more API keys:
   ```bash
   openssl rand -hex 32
   ```

**Optional Configuration**:

- **WEBHOOK_SECRET**: Set if you want to verify webhook signatures
- **EMAIL**: Email for Let's Encrypt notifications
- **IMAGE_TAG**: Specific version to deploy (default: `latest`)

### 3. Configure DNS

Point your domain to your server's IP address:

```
A Record: webhooks.example.com -> YOUR_SERVER_IP
```

Wait for DNS propagation (can take a few minutes to hours).

### 4. Deploy the Stack

```bash
# Pull the latest images
docker compose pull

# Start the services
docker compose up -d

# Check the logs
docker compose logs -f
```

### 5. Verify Deployment

Check that all services are running:

```bash
docker compose ps
```

Test the health endpoint:

```bash
curl https://webhooks.example.com/health
```

Expected response:
```json
{"status":"healthy","database":"connected"}
```

## Configuration Reference

### Environment Variables

See [.env.example](.env.example) for a complete list of all configuration options.

#### Critical Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `DOMAIN` | Yes | Your domain name | `webhooks.example.com` |
| `POSTGRES_PASSWORD` | Yes | Database password | `generate_with_openssl` |
| `API_KEYS` | Yes | Comma-separated API keys | `key1,key2,key3` |
| `EMAIL` | Recommended | Let's Encrypt email | `admin@example.com` |
| `WEBHOOK_SECRET` | Optional | YouTube webhook secret | `your_secret` |

#### Swag/SSL Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `VALIDATION` | `http` | Certificate validation method (`http` or `dns`) |
| `STAGING` | `false` | Use Let's Encrypt staging (for testing) |
| `HTTP_PORT` | `80` | HTTP port on host |
| `HTTPS_PORT` | `443` | HTTPS port on host |

## Usage

### Managing Services

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart a specific service
docker compose restart webhook-service

# View logs
docker compose logs -f webhook-service

# View all logs
docker compose logs -f
```

### Updating to Latest Version

```bash
# Pull the latest images
docker compose pull

# Recreate containers with new images
docker compose up -d

# Remove old images
docker image prune -f
```

### Managing Subscriptions

Use the `/api/v1/subscriptions` endpoint with your API key:

```bash
# Subscribe to a YouTube channel
curl -X POST https://webhooks.example.com/api/v1/subscriptions \
  -H "X-API-Key: your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_id": "UCuAXFkgsw1L7xaCfnd5JJOw",
    "callback_url": "https://webhooks.example.com/webhook",
    "lease_seconds": 432000
  }'

# List active subscriptions
curl -X GET https://webhooks.example.com/api/v1/subscriptions \
  -H "X-API-Key: your_api_key_here"
```

### Database Access

To access the PostgreSQL database directly:

```bash
# Connect to database
docker compose exec postgres psql -U postgres -d youtube_webhooks

# Or use a one-liner
docker compose exec postgres psql -U postgres -d youtube_webhooks -c "SELECT COUNT(*) FROM webhook_events;"
```

### Backups

#### Database Backup

```bash
# Create a backup
docker compose exec postgres pg_dump -U postgres youtube_webhooks > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore from backup
docker compose exec -T postgres psql -U postgres youtube_webhooks < backup.sql
```

#### Volume Backup

```bash
# Backup PostgreSQL data volume
docker run --rm -v youtube-webhook-postgres-data:/data -v $(pwd):/backup alpine tar czf /backup/postgres-data-$(date +%Y%m%d).tar.gz -C /data .

# Backup Swag config volume
docker run --rm -v youtube-webhook-swag-config:/data -v $(pwd):/backup alpine tar czf /backup/swag-config-$(date +%Y%m%d).tar.gz -C /data .
```

## Monitoring

### Health Checks

All services have built-in health checks:

```bash
# Check health of all services
docker compose ps

# Watch health status
watch docker compose ps
```

### Logs

```bash
# Follow all logs
docker compose logs -f

# Follow specific service logs
docker compose logs -f webhook-service
docker compose logs -f swag
docker compose logs -f postgres

# View webhook access logs
docker compose exec swag tail -f /config/log/nginx/webhook-access.log

# View webhook error logs
docker compose exec swag tail -f /config/log/nginx/webhook-error.log
```

### Metrics

Check service statistics:

```bash
# Container resource usage
docker stats

# Service-specific stats
docker stats youtube-webhook-service
```

## Troubleshooting

### SSL Certificate Issues

If SSL certificates aren't generated:

1. **Verify DNS**: Ensure your domain points to your server:
   ```bash
   dig +short webhooks.example.com
   nslookup webhooks.example.com
   ```

2. **Check Firewall**: Ensure ports 80 and 443 are open:
   ```bash
   sudo ufw status
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

3. **Use Staging**: Test with Let's Encrypt staging first:
   ```bash
   # In .env file
   STAGING=true
   ```

4. **Check Swag Logs**:
   ```bash
   docker compose logs swag
   ```

### Service Not Starting

```bash
# Check container status
docker compose ps

# View detailed logs
docker compose logs webhook-service

# Check for port conflicts
sudo netstat -tulpn | grep -E ':(80|443|5432)'

# Restart services
docker compose restart
```

### Database Connection Issues

```bash
# Verify database is healthy
docker compose exec postgres pg_isready -U postgres

# Check database logs
docker compose logs postgres

# Test connection from webhook service
docker compose exec webhook-service wget -O- http://localhost:8080/health
```

### Rate Limiting

If you're being rate limited:

1. Check the Nginx logs:
   ```bash
   docker compose exec swag tail -f /config/log/nginx/webhook-error.log
   ```

2. Adjust rate limits in `swag/nginx/site-confs/00-rate-limits.conf`

3. Restart Swag:
   ```bash
   docker compose restart swag
   ```

## Security Considerations

1. **Strong Passwords**: Always use strong, randomly generated passwords
2. **API Keys**: Generate secure API keys and rotate them regularly
3. **File Permissions**: Restrict `.env` file permissions:
   ```bash
   chmod 600 .env
   ```
4. **Webhook Secret**: Use `WEBHOOK_SECRET` to verify webhook signatures
5. **Firewall**: Only expose necessary ports (80, 443)
6. **Updates**: Keep Docker images updated regularly
7. **Monitoring**: Monitor logs for suspicious activity

## Advanced Configuration

### Custom Nginx Configuration

To customize Nginx settings:

1. Edit files in `swag/nginx/site-confs/`
2. Restart Swag: `docker compose restart swag`
3. Check for errors: `docker compose logs swag`

### DNS Validation (Recommended for Production)

For better security and wildcard certificates:

1. Set `VALIDATION=dns` in `.env`
2. Set your DNS provider in `DNSPLUGIN` (e.g., `cloudflare`)
3. Configure DNS provider credentials in Swag config volume
4. Restart: `docker compose up -d`

See [Swag documentation](https://docs.linuxserver.io/general/swag) for DNS provider setup.

### Using a Specific Version

To deploy a specific version instead of `latest`:

```bash
# In .env file
IMAGE_TAG=v1.2.3

# Pull and deploy
docker compose pull
docker compose up -d
```

## Uninstalling

To completely remove the deployment:

```bash
# Stop and remove containers
docker compose down

# Remove volumes (WARNING: This deletes all data!)
docker compose down -v

# Remove cloned repository
cd ..
rm -rf youtube-webhook-ingestion-deploy
```

## Support

- **Main Service Repository**: [youtube-webhook-ingestion-go](https://github.com/ad-tracker/youtube-webhook-ingestion-go)
- **Issues**: [Report an issue](https://github.com/ad-tracker/youtube-webhook-ingestion-deploy/issues)
- **Documentation**: See the main service repository for API documentation

## License

See the main service repository for license information.

## Contributing

Contributions are welcome! Please open an issue or pull request.
