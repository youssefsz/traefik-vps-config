# Traefik Reverse Proxy Configuration

A production-ready Traefik v3 reverse proxy configuration for managing multiple Docker-based services with automatic HTTPS via Let's Encrypt.

## Overview

This repository contains the configuration files for a Traefik reverse proxy deployment. Traefik acts as the central ingress point for all web traffic, handling SSL termination, load balancing, and routing to backend Docker containers.

## Features

- Automatic HTTPS certificates via Let's Encrypt (ACME TLS challenge)
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
├── acme.json               # Let's Encrypt certificates (auto-generated, not tracked)
└── .gitignore
```

## Prerequisites

- Docker and Docker Compose
- A domain name pointing to your server
- Port 80 and 443 available on the host

## Setup

### 1. Create the Traefik network

Before starting Traefik, create the external Docker network that all services will share:

```bash
docker network create traefik
```

### 2. Create the ACME storage file

Traefik requires a file to store SSL certificates. This file must have restricted permissions:

```bash
touch acme.json
chmod 600 acme.json
```

### 3. Start Traefik

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
      - "traefik.http.routers.your-service.tls.certresolver=myresolver"
      - "traefik.http.routers.your-service.middlewares=default-chain@file"
    networks:
      - traefik

networks:
  traefik:
    external: true
```

## Security Notes

- The `acme.json` file contains SSL private keys and should never be committed to version control
- The Docker socket is mounted read-only to limit potential security exposure
- Services are not exposed by default; they must explicitly set `traefik.enable=true`

## Troubleshooting

### Check Traefik logs

```bash
docker logs traefik
```

### Verify certificate status

Check that `acme.json` is being populated with certificates after Traefik processes requests to your domains.

### Common issues

1. **Certificate not issued**: Ensure port 443 is accessible from the internet and DNS is properly configured
2. **Service not discovered**: Verify the service is on the `traefik` network and has `traefik.enable=true`
3. **Permission denied on acme.json**: The file must have 600 permissions

## References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Docker Provider](https://doc.traefik.io/traefik/providers/docker/)

## License

This configuration is provided as-is for personal and commercial use.
