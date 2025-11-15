# Swag Configuration

This directory contains the Nginx configuration files for the Swag reverse proxy.

## Directory Structure

```
swag/
└── nginx/
    └── site-confs/
        ├── 00-rate-limits.conf     # Rate limiting configuration
        └── youtube-webhook.conf    # Main site configuration
```

## Configuration Files

### 00-rate-limits.conf

Defines rate limiting zones that are used across all site configurations:

- `webhook_limit`: 10 requests per second per IP for webhook endpoints
- `api_limit`: 5 requests per second per IP for API endpoints
- `conn_limit`: Connection limiting zone

This file is loaded first (due to the `00-` prefix) to ensure rate limiting zones are available to other configurations.

### youtube-webhook.conf

Main Nginx server configuration that:

1. **HTTPS Server Block (Port 443)**:
   - Handles all HTTPS traffic
   - Proxies requests to the webhook service
   - Implements security headers
   - Configures rate limiting for different endpoints

2. **HTTP Server Block (Port 80)**:
   - Redirects all HTTP traffic to HTTPS
   - Allows Let's Encrypt ACME challenge for certificate generation

## Endpoints

### Public Endpoints

- `/webhook` - YouTube webhook notifications (rate limited: 10 req/s)
- `/health` - Health check endpoint (no rate limiting)

### Protected Endpoints

- `/api/*` - API endpoints (rate limited: 5 req/s, requires API key)

## Rate Limiting

The configuration implements different rate limits for different endpoints:

- Webhook: 10 requests/second with burst of 20
- API: 5 requests/second with burst of 10
- Health check: No rate limiting

When rate limits are exceeded, the server returns HTTP 429 (Too Many Requests).

## Security Headers

The following security headers are automatically added to all responses:

- `X-Frame-Options: SAMEORIGIN`
- `X-Content-Type-Options: nosniff`
- `X-XSS-Protection: 1; mode=block`
- `Referrer-Policy: no-referrer-when-downgrade`

## Logs

Logs are stored in the Swag config volume:

- `/config/log/nginx/webhook-access.log` - Webhook access logs
- `/config/log/nginx/webhook-error.log` - Webhook error logs
- `/config/log/nginx/api-access.log` - API access logs
- `/config/log/nginx/api-error.log` - API error logs

## Customization

To customize the configuration:

1. Edit the files in `swag/nginx/site-confs/`
2. Restart the Swag container: `docker compose restart swag`
3. Check logs for any configuration errors: `docker compose logs swag`

## Note

The Swag container automatically handles SSL/TLS certificate generation and renewal using Let's Encrypt. Make sure your domain is properly configured and accessible from the internet for certificate validation.
