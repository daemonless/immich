# Immich Stack

Complete self-hosted photo and video management solution.

| | |
|---|---|
| **Port** | 2283 |
| **Registry** | `ghcr.io/daemonless/immich` |
| **Source** | [https://github.com/immich-app/immich](https://github.com/immich-app/immich) |
| **Website** | [https://immich.app/](https://immich.app/) |

## Deployment

### Podman Compose

```yaml
services:
  immich:
    image: ghcr.io/daemonless/immich:latest
    container_name: immich
    environment:
      - DB_HOSTNAME=immich_postgres
      - DB_USERNAME=postgres
      - DB_PASSWORD=${DB_PASSWORD:-postgres}
      - DB_DATABASE_NAME=immich
      - REDIS_HOSTNAME=immich_redis
      - PUID=1000
      - PGID=1000
      - TZ=UTC
    volumes:
      - /path/to/containers/immich:/config
      - /path/to/data:/data
    ports:
      - 2283:2283
    restart: unless-stopped
```

### Podman CLI

```bash
podman run -d --name immich \
  -p 2283:2283 \
  -e DB_HOSTNAME=immich_postgres \
  -e DB_USERNAME=postgres \
  -e DB_PASSWORD=${DB_PASSWORD:-postgres} \
  -e DB_DATABASE_NAME=immich \
  -e REDIS_HOSTNAME=immich_redis \
  -e PUID=@PUID@ \
  -e PGID=@PGID@ \
  -e TZ=@TZ@ \
  -v /path/to/containers/immich:/config \ 
  -v /path/to/data:/data \ 
  ghcr.io/daemonless/immich:latest
```
Access at: `http://localhost:2283`

### Ansible

```yaml
- name: Deploy immich
  containers.podman.podman_container:
    name: immich
    image: ghcr.io/daemonless/immich:latest
    state: started
    restart_policy: always
    env:
      DB_HOSTNAME: "immich_postgres"
      DB_USERNAME: "postgres"
      DB_PASSWORD: "${DB_PASSWORD:-postgres}"
      DB_DATABASE_NAME: "immich"
      REDIS_HOSTNAME: "immich_redis"
      PUID: "1000"
      PGID: "1000"
      TZ: "UTC"
    ports:
      - "2283:2283"
    volumes:
      - "/path/to/containers/immich:/config"
      - "/path/to/data:/data"
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOSTNAME` | `immich_postgres` |  |
| `DB_USERNAME` | `postgres` |  |
| `DB_PASSWORD` | `${DB_PASSWORD:-postgres}` | Postgres database password |
| `DB_DATABASE_NAME` | `immich` |  |
| `REDIS_HOSTNAME` | `immich_redis` |  |
| `PUID` | `1000` | User ID for application processes |
| `PGID` | `1000` | Group ID for application processes |
| `TZ` | `UTC` | Timezone |

### Volumes

| Path | Description |
|------|-------------|
| `/config` | Configuration files |
| `/data` | Media storage (mapped to UPLOAD_LOCATION) |

### Ports

| Port | Protocol | Description |
|------|----------|-------------|
| `2283` | TCP | Web UI |

## Notes

- **User:** `multiple` (UID/GID set via PUID/PGID)
- **Base:** Built on `ghcr.io/daemonless/base` (FreeBSD)