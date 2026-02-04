# VPC Nginx Reverse Proxy - Project Context

This document provides an overview of the Nginx configuration, its role in the VPC stack, and its deployment details.

## Overview

The `vpc-nginx` service acts as the central entry point for all external traffic to the VPC stack. It handles SSL termination (via Certbot), reverse proxying to various backends, and path-based routing.

### Key Features

- **Runtime:** `nginx:1.29.4-alpine`
- **SSL Management:** Uses `inotify-tools` to monitor `/etc/letsencrypt/live` and automatically reload Nginx (`nginx -s reload`) when certificates are updated.
- **Security:** Implements standard security headers (HSTS, XSS protection) and secure cookie flags on the public edge.
- **Dynamic Routing:** Uses Nginx variables and `resolver 127.0.0.11` to handle Docker internal DNS resolution dynamically.

## Configuration Structure

The configuration is modular and organized within `/etc/nginx/`:

- `nginx.conf`: The main entry point. Loads server blocks based on the environment (e.g., `servers-public.conf` for production, `servers-dev.conf` for local).
- `conf.d/`: Contains environment-specific server blocks:
  - `servers-public.conf`: Production config with SSL (ports 443).
  - `servers-dev.conf`: Local development config (port 80).
  - `servers-aws.conf`: Specific config for AWS EC2 instances.
- `conf.d/includes/`: Reusable configuration snippets:
  - `upstream-vars.conf`: Defines variables for backend services (e.g., `$vpc_api`, `$vpc_next`).
  - `routes-vpc.conf`: Routing for `/vpc/*` (includes CORS for `vpinhub.github.io`).
  - `routes-vps.conf`: Routing for `/vps/*` to the `vpc-data` service.
  - `proxy-common.conf`: Shared proxy settings (timeouts, headers).
  - `bot-dropper.conf`: Logic to drop unwanted bot traffic.

## Development & Deployment

### Building

```bash
docker build -t vpc-nginx .
```

### SSL Certificates

In production, certificates are mounted from the `certbot` service via a shared volume. The Dockerfile's `CMD` ensures that Nginx reloads automatically whenever Certbot renews a certificate.

## Important Notes

- **CORS:** The `/vpc/` route has explicit CORS support for `https://vpinhub.github.io`.
- **User:** Nginx runs as the `www-data` user for improved security.
- **Timeouts:** `proxy_read_timeout` is set to 300s to accommodate long-running API requests if necessary.
