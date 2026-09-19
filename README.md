# Traefik Reverse Proxy Configuration

A production-ready Traefik v3 reverse proxy configuration for managing multiple Docker-based services with automatic HTTPS via Let's Encrypt.

## Overview

This repository contains the configuration files for a Traefik reverse proxy deployment. Traefik acts as the central ingress point for all web traffic, handling SSL termination, load balancing, and routing to backend Docker containers.

## Features

- Automatic HTTPS certificates via Let's Encrypt (Cloudflare DNS-01 challenge)
- HTTP to HTTPS redirection
- Docker provider integration with automatic service discovery
- Reusable middleware configurations (rate limiting, security headers, compression)
- Resource limits for container stability

## Directory Structure

```
traefik/
├── docker-compose.yml      # Main Traefik service definition
├── dynamic/
│   └── middlewares.yml     # Reusable middleware configurations
├── acme.json               # Existing TLS-challenge certificates (not tracked)
├── acme-cloudflare.json    # Cloudflare DNS-challenge certificates (not tracked)
└── .gitignore
```

## Prerequisites

- Docker and Docker Compose
- A domain name pointing to your server
- Port 80 and 443 available on the host
- A Cloudflare API token scoped to `Zone:Read` and `DNS:Edit` for the zones Traefik manages

## Setup

### 1. Create the Traefik network

Before starting Traefik, create the external Docker network that all services will share:

```bash
docker network create traefik
```

### 2. Create the ACME storage file

Traefik uses separate storage files for the existing TLS-challenge resolver and the Cloudflare DNS-challenge resolver. Both files must have restricted permissions:

```bash
touch acme.json
touch acme-cloudflare.json
chmod 600 acme.json
chmod 600 acme-cloudflare.json
```

### 3. Add the Cloudflare DNS API token

Create a scoped Cloudflare API token with these permissions:

- `Zone / Zone / Read`
- `Zone / DNS / Edit`

Restrict the token to the zones served by this Traefik instance. Store it as a Docker secret:

```bash
mkdir -p secrets
chmod 700 secrets
printf '%s' 'YOUR_TOKEN' > secrets/cloudflare_dns_api_token
chmod 600 secrets/cloudflare_dns_api_token
```

The `secrets/` directory is ignored by Git. Never commit this token.

### 4. Start Traefik

```bash
docker compose up -d
```

## Configuration Details

### Docker Compose

The `docker-compose.yml` file defines the Traefik service with the following key settings:

- **Ports**: 80 (HTTP) and 443 (HTTPS)
- **Volumes**: Docker socket (read-only), ACME storage, and dynamic configuration directory
- **Security**: `no-new-privileges` security option enabled
- **Resources**: Limited to 0.5 CPU and 512MB memory

### Middlewares

The `dynamic/middlewares.yml` file contains reusable middleware configurations:

#### Rate Limiting

- `rate-limit`: Standard rate limiting (100 req/s average, 200 burst)
- `rate-limit-strict`: Stricter rate limiting for sensitive endpoints (20 req/s average, 50 burst)

#### Security Headers

The `secure-headers` middleware adds the following protections:

- Frame denial (clickjacking protection)
- Content type sniffing prevention
- XSS filter
- HSTS with 1-year duration and preload
- Content Security Policy
- Referrer policy

#### Compression

The `compress` middleware enables gzip compression for responses.

#### Middleware Chains

Pre-configured chains for convenience:

- `default-chain`: Rate limit + security headers + compression
- `strict-chain`: Strict rate limit + security headers + compression

## Adding a New Service

To add a new service behind Traefik, add the following labels to your Docker Compose service:

```yaml
services:
  your-service:
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.your-service.rule=Host(`your-domain.com`)"
      - "traefik.http.routers.your-service.entrypoints=websecure"
      - "traefik.http.routers.your-service.tls.certresolver=cloudflare"
      - "traefik.http.routers.your-service.middlewares=default-chain@file"
    networks:
      - traefik

networks:
  traefik:
    external: true
```

## Security Notes

- The ACME JSON files contain SSL private keys and should never be committed to version control
- The Docker socket is mounted read-only to limit potential security exposure
- Services are not exposed by default; they must explicitly set `traefik.enable=true`

## Troubleshooting

### Check Traefik logs

```bash
docker logs traefik
```

### Verify certificate status

Check that `acme-cloudflare.json` is populated after Traefik processes a router that uses the `cloudflare` resolver. Existing routers using `myresolver` continue to use `acme.json`.

### Common issues

1. **Certificate not issued**: Verify the Cloudflare token has `Zone:Read` and `DNS:Edit` access to the requested domain
2. **Service not discovered**: Verify the service is on the `traefik` network and has `traefik.enable=true`
3. **Permission denied on an ACME file**: Both ACME JSON files must have 600 permissions

## References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Docker Provider](https://doc.traefik.io/traefik/providers/docker/)

## License

This configuration is provided as-is for personal and commercial use.
