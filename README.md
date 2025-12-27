# Immich for FreeBSD

Native FreeBSD port of [Immich](https://immich.app/) - the self-hosted photo and video management solution.

**Drop-in compatible** with official Linux Immich. Your existing data works unchanged.

## Containers

| Container | Image | Purpose |
|-----------|-------|---------|
| [immich-server](https://github.com/daemonless/immich-server) | `ghcr.io/daemonless/immich-server` | Main application (Node.js) |
| [immich-postgres](https://github.com/daemonless/immich-postgres) | `ghcr.io/daemonless/immich-postgres` | Database (PostgreSQL 14 + VectorChord) |
| [redis](https://github.com/daemonless/redis) | `ghcr.io/daemonless/redis` | Redis (FreeBSD Package) |

## Machine Learning (Important)

FreeBSD does not currently support the Immich Machine Learning container due to missing `onnxruntime` Python bindings.

**Workaround:** Run the official `immich-machine-learning` container on a Linux host (or Linux VM/Jail) and point your FreeBSD server to it.

```bash
# In .env
IMMICH_MACHINE_LEARNING_URL=http://<linux-host-ip>:3003
```

## Quick Start (Compose)

1.  **Download `.env`** from official Immich:
    ```bash
    fetch https://github.com/immich-app/immich/releases/latest/download/.env
    ```

2.  **Download `docker-compose.yml`**:
    ```bash
    fetch https://raw.githubusercontent.com/daemonless/immich/main/docker-compose.yml
    ```

3.  **Start:**
    ```bash
    podman-compose up -d
    ```

## Volumes & Compatibility

| Official Volume | Daemonless Path | Notes |
|-----------------|-----------------|-------|
| `pgdata` | `/config` | PostgreSQL data is stored in `/config/data` inside the container. Mapping works transparently. |
| `redis-data` | `/config` | Redis data is stored in `/config/data`. |
| `UPLOAD_LOCATION` | `/usr/src/app/upload` | Standard upload path. |

## Migration

Migration from Linux to FreeBSD is seamless:

1.  Stop Linux containers.
2.  Copy your `library`, `pgdata`, and `redis-data` volumes to the FreeBSD host.
3.  Update your `.env` `UPLOAD_LOCATION` to match the new path.
4.  Start the Daemonless containers.

## Links

- [Immich Documentation](https://immich.app/docs)
- [Daemonless Project](https://github.com/daemonless)