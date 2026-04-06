# docker-guard Proxy Reference

## What it does

docker-guard is a TCP proxy between hermes-agent and the Docker daemon that enforces
label-based access control. It runs as a sidecar container on port 2375.

## Access Control Matrix

| Docker API | Method | Policy |
|------------|--------|--------|
| `GET` (ps, inspect, logs, version, info) | GET/HEAD | **Allowed** — read-only, no side effects |
| `POST /containers/create` | POST | **Allowed** — `hermes-test=true` label auto-injected |
| `POST /containers/{id}/start\|stop\|restart\|kill` | POST | **Allowed** only if `hermes-test=true` or name starts with `hermes-` |
| `DELETE /containers/{id}` | DELETE | **Allowed** only if `hermes-test=true` or name starts with `hermes-` |
| `POST /containers/{id}/exec` | POST | **Allowed** only if `hermes-test=true` or name starts with `hermes-` |
| `POST /networks/{id}/connect\|disconnect` | POST | **Allowed** only if the container in the body passes the label check |
| `DELETE /networks/{id}` | DELETE | **Blocked** — prevents accidental removal of `traefik-public` |
| `POST */prune` | POST | **Blocked** — no prune operations |
| `POST /images/create` (pull) | POST | **Allowed** |
| `POST /volumes/create` | POST | **Allowed** |

## Label Injection

When hermes creates a container via `POST /containers/create`, docker-guard parses the
JSON body and injects:

```json
{"Labels": {"hermes-test": "true"}}
```

This means even if the agent forgets the label, the proxy adds it. All subsequent
mutation operations on that container will be allowed.

## Name Prefix Exception

Containers whose names start with `hermes-` (the agent's sandbox containers) are always
allowed, even without the `hermes-test` label. This is needed because the agent's own
`DockerEnvironment` creates sandbox containers with names like `hermes-a1b2c3d4`.

## Limitation: Docker Exec Hijack

The proxy uses `Connection: close` for all upstream requests. This means Docker's
interactive exec hijack protocol (HTTP 101 upgrade to raw TCP) does NOT work through
the proxy. For running commands inside containers, use the `local` terminal backend
or non-interactive `docker exec` without `-it`.

## Network Architecture

```
hermes container (DOCKER_HOST=tcp://docker-guard:2375)
    ↓
docker-guard container (port 2375, has docker:ro socket)
    ↓
/var/run/docker.sock (host Docker daemon)
```

Both containers are on the `vless-net` network. docker-guard does NOT expose port 2375
to the host — it is only reachable from within the docker-compose stack.
