# Immich

Native FreeBSD port of [Immich](https://immich.app/) - the self-hosted photo and video management solution.

**Drop-in compatible** with official Linux Immich. Your existing data works unchanged.

## Architecture

This stack is composed of the following specialized FreeBSD containers:

| Service | Container Image | Description |
|---------|-----------------|-------------|
| **Server** | [`ghcr.io/daemonless/immich-server`](https://github.com/daemonless/immich-server) | Main Node.js application (Web/API) |
| **Database** | [`ghcr.io/daemonless/immich-postgres`](https://github.com/daemonless/immich-postgres) | PostgreSQL 14 with `pgvecto.rs` extension |
| **Redis** | [`ghcr.io/daemonless/redis`](https://github.com/daemonless/redis) | Redis cache (FreeBSD package) |
| **Machine Learning** | *External / Experimental* | See below |

## Quick Start (Compose)

1.  **Download `.env`** from official Immich releases:
    ```bash
    fetch https://github.com/immich-app/immich/releases/latest/download/.env
    ```

2.  **Download `docker-compose.yml`**:
    ```bash
    fetch https://raw.githubusercontent.com/daemonless/immich/main/docker-compose.yml
    ```

3.  **Configure `.env`**:
    Edit the file and set `UPLOAD_LOCATION` to your desired storage path.

4.  **Start the Stack**:
    ```bash
    podman-compose up -d
    ```

    Access the web interface at: http://localhost:2283

## Environment Variables

The stack relies on the standard Immich `.env` file. Key variables include:

| Variable | Description | Default |
|----------|-------------|---------|
| `UPLOAD_LOCATION` | Path on host to store photos/videos | `./library` |
| `DB_PASSWORD` | PostgreSQL password | `postgres` |
| `DB_USERNAME` | PostgreSQL user | `postgres` |
| `DB_DATABASE_NAME` | PostgreSQL database name | `immich` |
| `IMMICH_MACHINE_LEARNING_URL` | URL to ML service | *Required if external* |

## Ports

| Port | Service | Description |
|------|---------|-------------|
| `2283` | Server | Main Web UI and API |

## Volumes

| Path (Host) | Container Path | Description |
|-------------|----------------|-------------|
| `${UPLOAD_LOCATION}` | `/usr/src/app/upload` | Main media library |
| `pgdata` (Volume) | `/config/data` | Database files |
| `redis-data` (Volume) | `/config/data` | Redis persistence |

## Machine Learning

FreeBSD does not currently support the Immich Machine Learning container natively due to missing `onnxruntime` Python bindings for FreeBSD.

**Workaround:**
Run the official `immich-machine-learning` container on a Linux host (or Linux VM/Jail) and point your FreeBSD server to it via `.env`:

```bash
IMMICH_MACHINE_LEARNING_URL=http://<linux-host-ip>:3003
```

## Logging

Each container in this stack uses `s6-log` for log management:
- **Location:** Logs are stored inside each container at `/config/logs/`.
- **Access:** View logs via `podman logs -f <container_name>`.

## Migration from Linux

Migration is seamless as data formats are identical:

1.  Stop the Linux containers.
2.  Copy your `library`, `pgdata`, and `redis-data` volumes to the FreeBSD host.
3.  Update `UPLOAD_LOCATION` in `.env` to match the new FreeBSD path.
4.  Start the Daemonless containers.

## Links

- [Official Website](https://immich.app/)
- [Documentation](https://immich.app/docs)
- [GitHub Upstream](https://github.com/immich-app/immich)
