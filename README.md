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
| **Machine Learning** | [`ghcr.io/daemonless/immich-ml`](https://github.com/daemonless/immich-ml) | Native ML service (CPU only) |

## Prerequisites

```bash
pkg install podman-suite cni-dnsname py311-podman-compose
```

- `cni-dnsname` enables DNS resolution between containers
- `py311-podman-compose` runs the compose file

## Quick Start (Compose)

1.  **Create directory and download files**:
    ```bash
    mkdir -p /containers/immich
    cd /containers/immich
    fetch https://raw.githubusercontent.com/daemonless/immich/main/container-compose.yml -o container-compose.yml
    ```

2.  **Create `.env` file**:
    ```bash
    cat > .env << 'EOF'
    # Immich configuration for FreeBSD
    # IMPORTANT: Use absolute paths, not relative paths

    UPLOAD_LOCATION=/containers/immich/library
    DB_PASSWORD=changeme
    DB_USERNAME=postgres
    DB_DATABASE_NAME=immich
    EOF
    ```

3.  **Create library directory**:
    ```bash
    mkdir -p /containers/immich/library
    chown 1000:1000 /containers/immich/library
    ```

4.  **Start the Stack**:
    ```bash
    podman-compose up -d
    ```

    Access the web interface at: http://localhost:2283

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `UPLOAD_LOCATION` | **Absolute path** to store photos/videos | `/containers/immich/library` |
| `DB_PASSWORD` | PostgreSQL password (change this!) | `changeme` |
| `DB_USERNAME` | PostgreSQL user | `postgres` |
| `DB_DATABASE_NAME` | PostgreSQL database name | `immich` |

**Important**: `UPLOAD_LOCATION` must be an absolute path (e.g., `/containers/immich/library`), not a relative path like `./library`.

## Ports

| Port | Service | Description |
|------|---------|-------------|
| `2283` | Server | Main Web UI and API |

## Volumes

| Path (Host) | Container Path | Description |
|-------------|----------------|-------------|
| `${UPLOAD_LOCATION}` | `/data` | Main media library |
| `pgdata` (Volume) | `/config/data` | Database files |
| `redis-data` (Volume) | `/config/data` | Redis persistence |

## Machine Learning

Immich Machine Learning is natively supported on FreeBSD.

**Note**: Machine Learning on FreeBSD currently runs on the **CPU only**. Hardware acceleration (GPU/NPU) is not yet supported.

### GPU Acceleration (Remote ML)

For faster ML inference, you can run the official Immich ML container on a Linux host with GPU acceleration and point your FreeBSD server to it.

**Supported GPUs:**
- **Intel** (OpenVINO): Core Ultra, Arc, 11th gen+
- **NVIDIA** (CUDA): Most modern GPUs

**1. Deploy ML on Linux host with Intel GPU:**

```bash
# On Linux host (e.g., Fedora, Ubuntu)
sudo podman run -d \
  --name immich_machine_learning \
  --device /dev/dri:/dev/dri \
  -v immich-ml-cache:/cache \
  -e MACHINE_LEARNING_WORKERS=1 \
  -p 3003:3003 \
  --restart unless-stopped \
  ghcr.io/immich-app/immich-machine-learning:release-openvino
```

For NVIDIA GPUs, use `ghcr.io/immich-app/immich-machine-learning:release-cuda` with `--gpus all`.

**2. Configure FreeBSD server to use remote ML:**

Add to your `.env` file:
```bash
IMMICH_MACHINE_LEARNING_URL=http://<linux-host-ip>:3003
```

**3. Restart the stack:**
```bash
podman-compose down && podman-compose up -d
```

The server will now use the GPU-accelerated ML service on your Linux host.

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
