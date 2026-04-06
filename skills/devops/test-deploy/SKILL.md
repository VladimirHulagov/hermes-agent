---
name: test-deploy
description: Deploy isolated test services via docker compose with Traefik integration. All containers are auto-labeled hermes-test=true and protected by docker-guard proxy.
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [docker, compose, traefik, test, deploy, infrastructure]
---

# Test Service Deployment

Deploy ephemeral test services that connect to the production Traefik reverse proxy
but are fully isolated from production containers via the docker-guard label proxy.

## Architecture

```
hermes-agent → docker-guard:2375 (label proxy) → /var/run/docker.sock
                                                         ↓
                                              Docker daemon
                                                         ↓
                                              traefik (reads labels on all containers)
```

- docker-guard auto-injects `hermes-test=true` label on every `POST /containers/create`
- docker-guard blocks stop/rm/exec on containers WITHOUT that label
- Traefik discovers test services via Docker provider labels on the same network

## Rules — STRICT

1. All compose files go to `/mnt/services/tests/<name>/docker-compose.yml`
2. All container names MUST start with `test-`
3. All subdomains use pattern `*.test.${DOMAIN}`
4. `restart` MUST be `"no"` — test services must not survive host reboot
5. Every service MUST include `traefik.enable: true` label
6. Every service MUST include basicauth middleware
7. NEVER use `restart: unless-stopped` or `restart: always`
8. NEVER mount `/var/run/docker.sock` into test containers
9. NEVER use production container names, networks, or volumes
10. NEVER modify files outside `/mnt/services/tests/`
11. **By default use `lan-macvlan` network (internal/LAN). Only use `traefik-public` (external/Internet) when explicitly requested by the user.**

## Compose File Template — Default (LAN / Internal)

Use this template by default. Service gets a LAN IP via macvlan, reachable from
the local network but NOT exposed to the Internet through Traefik.

```yaml
# /mnt/services/tests/<name>/docker-compose.yml
version: "3"

services:
  app:
    image: <IMAGE>
    container_name: test-<name>
    restart: "no"
    networks:
      - lan
    volumes:
      - test-<name>-data:/app/data

volumes:
  test-<name>-data:

networks:
  lan:
    name: lan-macvlan
    external: true
```

## Compose File Template — External (Traefik / Internet)

Use ONLY when the user explicitly asks for external/Internet access.
Service is reachable via `*.test.${DOMAIN}` through Traefik with TLS and basicauth.

```yaml
# /mnt/services/tests/<name>/docker-compose.yml
version: "3"

services:
  app:
    image: <IMAGE>
    container_name: test-<name>
    restart: "no"
    networks:
      - traefik-public
    labels:
      traefik.enable: true
      traefik.http.routers.test-<name>.rule: Host(`<subdomain>.test.${DOMAIN}`)
      traefik.http.routers.test-<name>.entrypoints: websecure
      traefik.http.routers.test-<name>.tls.certresolver: myresolver
      traefik.http.routers.test-<name>.middlewares: test-<name>-auth
      traefik.http.middlewares.test-<name>-auth.basicauth.users: ${TEST_BASIC_AUTH}
    volumes:
      - test-<name>-data:/app/data

volumes:
  test-<name>-data:

networks:
  traefik-public:
    external: true
```

## Combined Template — Both LAN and External

When a service needs both LAN access AND external Traefik routing:

```yaml
# /mnt/services/tests/<name>/docker-compose.yml
version: "3"

services:
  app:
    image: <IMAGE>
    container_name: test-<name>
    restart: "no"
    networks:
      - lan
      - traefik-public
    labels:
      traefik.enable: true
      traefik.http.routers.test-<name>.rule: Host(`<subdomain>.test.${DOMAIN}`)
      traefik.http.routers.test-<name>.entrypoints: websecure
      traefik.http.routers.test-<name>.tls.certresolver: myresolver
      traefik.http.routers.test-<name>.middlewares: test-<name>-auth
      traefik.http.middlewares.test-<name>-auth.basicauth.users: ${TEST_BASIC_AUTH}
    volumes:
      - test-<name>-data:/app/data

volumes:
  test-<name>-data:

networks:
  lan:
    name: lan-macvlan
    external: true
  traefik-public:
    external: true
```

## Commands

### Deploy a test service

```bash
mkdir -p /mnt/services/tests/<name>
# Write docker-compose.yml to the directory
docker compose -f /mnt/services/tests/<name>/docker-compose.yml up -d
```

### Check status

```bash
docker compose -f /mnt/services/tests/<name>/docker-compose.yml ps
```

### View logs

```bash
docker compose -f /mnt/services/tests/<name>/docker-compose.yml logs -f --tail=100
```

### Tear down (remove containers + volumes)

```bash
docker compose -f /mnt/services/tests/<name>/docker-compose.yml down -v
```

### Clean up all test services

```bash
for dir in /mnt/services/tests/*/; do
  docker compose -f "${dir}docker-compose.yml" down -v 2>/dev/null
done
```

## Traefik Labels Quick Reference

| Label | Purpose |
|-------|---------|
| `traefik.enable` | Must be `true` |
| `traefik.http.routers.<name>.rule` | Host matching rule |
| `traefik.http.routers.<name>.entrypoints` | Usually `websecure` |
| `traefik.http.routers.<name>.tls.certresolver` | `myresolver` (LetsEncrypt) |
| `traefik.http.routers.<name>.middlewares` | Middleware chain |
| `traefik.http.middlewares.<name>-auth.basicauth.users` | `user:$$hash` format |
| `traefik.http.services.<name>.loadbalancer.server.port` | Internal container port |

## Environment Variables

| Variable | Source | Purpose |
|----------|--------|---------|
| `DOMAIN` | traefik `.env` | Base domain (e.g. `example.com`) |
| `TEST_BASIC_AUTH` | Must be provided | htpasswd string for test auth |

Generate basicauth string:
```bash
echo $(htpasswd -nb user password) | sed -e 's/\$/\$\$/g'
```

## Troubleshooting

- **Container not reachable on LAN**: verify `lan-macvlan` network exists (`docker network ls`), check container got an IP (`docker inspect test-<name> | grep IPAddress`)
- **Container not reachable via Traefik**: verify `traefik-public` network exists, verify labels with `docker inspect test-<name>`
- **TLS cert error**: check that `*.test.DOMAIN` is covered by the wildcard cert in traefik (DNS challenge + SANs)
- **502 Bad Gateway**: the container port may not match `loadbalancer.server.port` label, or the app isn't ready yet
- **docker-guard blocks operation**: you tried to operate on a non-test container — only containers with `hermes-test=true` label or `hermes-` name prefix are mutable
