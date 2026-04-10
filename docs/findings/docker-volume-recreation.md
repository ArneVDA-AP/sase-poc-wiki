---
title: "Finding: Docker volume mounts require container recreation"
tags: [finding, workaround]
---

# Finding: Docker volume mounts require container recreation

**Component:** [Caddy](../components/caddy.md), [NetBird](../components/netbird.md)  
**Severity:** Gotcha

## What happened

After adding a new volume mount to the `caddy` service in `docker-compose.yml`, running `docker compose restart caddy` did not make the new mount appear inside the container. The expected file path was not accessible and the service continued behaving as before the change.

## Root cause

`docker compose restart` sends SIGTERM then SIGKILL to the container process and restarts it — but it reuses the existing container configuration, including the volume mounts that were in place when the container was first created. New volume mounts in `docker-compose.yml` are only applied when the container is recreated.

## Resolution / workaround

Use `docker compose up -d <service>` instead of `docker compose restart <service>`:

```bash
docker compose up -d caddy
```

`docker compose up -d` detects that the container configuration has changed (due to the new volume mount) and recreates the container with the updated configuration.

## Lessons

- `docker compose restart` ≠ "apply config changes" — it only restarts the existing container
- `docker compose up -d` applies configuration changes by recreating containers where needed
- This is a common Docker Compose misconception that wastes significant debugging time — when a config change appears not to take effect, always try `up -d` before assuming the config is wrong
