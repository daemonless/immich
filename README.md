---
title: "Immich Stack on FreeBSD: Native Photo Management using Podman & Jails"
description: "Deploy the complete Immich photo management stack on FreeBSD natively using Podman and Daemonless. Self-hosted Google Photos alternative with facial recognition and machine learning."
---

# :simple-googlephotos: Immich Stack

[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/immich-server/build.yaml?style=flat-square&label=Server&color=green)](https://github.com/daemonless/immich-server/actions)
[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/immich-ml/build.yaml?style=flat-square&label=ML&color=green)](https://github.com/daemonless/immich-ml/actions)
[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/immich-postgres/build.yaml?style=flat-square&label=Postgres&color=green)](https://github.com/daemonless/immich-postgres/actions)

Self-hosted photo and video management solution with facial recognition, object detection, and smart search - running natively on FreeBSD.

!!! tip "What is Immich?"
    Immich is a high-performance, self-hosted alternative to Google Photos. It provides automatic backup from mobile devices, facial recognition, object detection, location-based browsing, and a modern web interface. The Daemonless stack runs entirely on native FreeBSD - no Linux emulation required.

## Architecture

The Immich stack consists of four containers:

| Service | Image | Description |
|---------|-------|-------------|
| **Server** | `ghcr.io/daemonless/immich-server` | Main application server (Node.js) - Web UI and API |
| **Machine Learning** | `ghcr.io/daemonless/immich-ml` | Python/ONNX service for facial recognition and smart search |
| **PostgreSQL** | `ghcr.io/daemonless/immich-postgres` | Database with pgvector extension for ML embeddings |
| **Redis** | `ghcr.io/daemonless/redis` | Cache and job queue |

!!! warning "PostgreSQL Version"
    `immich-postgres:latest` and `:14` are both **PostgreSQL 14** — the current default, matching upstream Immich.
    New installs can use `:18` (PostgreSQL 18). Upgrading an existing database requires a full dump/restore.
    Upstream Immich is working on a migration path; we'll update these docs when that lands.

```mermaid
flowchart TD
    Client[Mobile / Web Client]
    Client -->|:2283| Server

    subgraph stack [Immich Stack]
        Server[immich-server<br/>Web UI + API]
        Server -->|:3003| ML[immich-ml<br/>Python/ONNX]
        Server -->|:5432| DB[immich-postgres<br/>pgvector]
        Server -->|:6379| Redis[redis<br/>cache]
    end

    click Server "immich-server.md" "immich-server"
    click ML "immich-ml.md" "immich-ml"
    click DB "immich-postgres.md" "immich-postgres"
    click Redis "redis.md" "redis"
```

## Prerequisites

Before deploying, ensure your host environment is ready. See the [Quick Start Guide](../guides/quick-start.md) for host setup instructions.

**Requirements:**

- FreeBSD 15.0 with Podman and ocijail
- `podman-compose`
- At least 4GB RAM (ML service is memory-intensive)
- Storage for photos (plan for growth)

## Deploy

=== ":material-tune: Podman Compose"

    !!! warning "Requires a patched ocijail"
        Immich needs `allow.mlock` and `allow.sysvipc` jail parameters. See the [ocijail patch guide](/guides/ocijail-patch/) before deploying.

    **1.** Save as `.env`:

    ```env
    # The location where your uploaded files are stored
    UPLOAD_LOCATION=./library

    # The location where your database files are stored
    DB_DATA_LOCATION=./postgres

    # To set a timezone, uncomment and change Etc/UTC to a TZ identifier
    # TZ=Etc/UTC

    # Change to a random password (A-Za-z0-9 only)
    DB_PASSWORD=postgres

    # The values below do not need to be changed
    ###################################################################################
    DB_USERNAME=postgres
    DB_DATABASE_NAME=immich
    ```

    **2.** Save as `compose.yaml`:

    ```yaml
    name: immich

    services:
      immich-server:
        container_name: immich_server
        image: ghcr.io/daemonless/immich-server:latest
        network_mode: host
        volumes:
          - ${UPLOAD_LOCATION}:/data
          - /etc/localtime:/etc/localtime:ro
        environment:
          DB_HOSTNAME: localhost
          REDIS_HOSTNAME: localhost
          IMMICH_MACHINE_LEARNING_URL: http://localhost:3003
        env_file:
          - .env
        depends_on:
          - redis
          - database
          - immich-machine-learning
        restart: always

      immich-machine-learning:
        container_name: immich_machine_learning
        image: ghcr.io/daemonless/immich-ml:latest
        network_mode: host
        environment:
          HF_HOME: /cache/huggingface  # ML models are cached here
          MPLCONFIGDIR: /tmp
        volumes:
          - model-cache:/cache
        env_file:
          - .env
        restart: always

      redis:
        container_name: immich_redis
        image: ghcr.io/daemonless/redis:latest
        network_mode: host
        volumes:
          - /etc/localtime:/etc/localtime:ro
          - redis-data:/config
        restart: always

      database:
        container_name: immich_postgres
        image: ghcr.io/daemonless/immich-postgres:14  # New installs can use :18
        network_mode: host
        annotations:
          org.freebsd.jail.allow.sysvipc: "true"
        environment:
          POSTGRES_PASSWORD: ${DB_PASSWORD}
          POSTGRES_USER: ${DB_USERNAME}
          POSTGRES_DB: ${DB_DATABASE_NAME}
        volumes:
          - /etc/localtime:/etc/localtime:ro
          - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
        restart: always

    volumes:
      model-cache:
      redis-data:
    ```

    **3.** Deploy:

    ```bash
    mkdir -p library postgres
    chown -R 1000:1000 library postgres
    podman-compose up -d
    ```

=== ":appjail-appjail: AppJail Director"

    **.env**:

    ```
    UPLOAD_LOCATION=/var/appjail-volumes/immich/library
    DB_DATA_LOCATION=/var/appjail-volumes/immich/postgres
    CACHE_LOCATION=/var/appjail-volumes/immich/cache
    REDIS_DATA_LOCATION=/var/appjail-volumes/immich/redis
    TZ=America/Caracas
    DB_PASSWORD=postgres
    DB_USERNAME=postgres
    DB_DATABASE_NAME=immich
    DIRECTOR_PROJECT=immich
    ```

    **appjail-director.yml**:

    ```yaml
    options:
      # Equivalent to 'network_mode: host'
      - alias:
      - ip4_inherit:
    services:
      immich-server:
        name: immich_server
        priority: 100
        options:
          - from: ghcr.io/daemonless/immich-server:latest
        volumes:
          - immich-data: /data
        oci:
          environment:
            - DB_HOSTNAME: 127.0.0.1
            - REDIS_HOSTNAME: 127.0.0.1
            - IMMICH_MACHINE_LEARNING_URL: http://127.0.0.1:3003
            - TZ: !ENV '${TZ}'
      immich-machine-learning:
        name: immich_machine_learning
        options:
          - from: ghcr.io/daemonless/immich-ml:latest
        oci:
          environment:
            - HF_HOME: /cache/huggingface
            - MPLCONFIGDIR: /tmp
            - TZ: !ENV '${TZ}'
        volumes:
          - model-cache: /cache
      redis:
        name: immich_redis
        options:
          - from: ghcr.io/daemonless/redis:latest
        oci:
          environment:
            - LANG: C.UTF-8
            - TZ: !ENV '${TZ}'
        volumes:
          - redis-data: /config
      database:
        name: immich_postgres
        options:
          - from: ghcr.io/daemonless/immich-postgres:latest
          - template: !ENV '${PWD}/immich-postgres-template.conf'
        oci:
          environment:
            - POSTGRES_PASSWORD: !ENV '${DB_PASSWORD}'
            - POSTGRES_USER: !ENV '${DB_USERNAME}'
            - POSTGRES_DB: !ENV '${DB_DATABASE_NAME}'
        volumes:
          - db-data: /var/lib/postgresql/data
    volumes:
      immich-data:
        device: !ENV '${UPLOAD_LOCATION}'
        owner: 1000
        group: 1000
      model-cache:
        device: !ENV '${CACHE_LOCATION}'
      redis-data:
        device: !ENV '${REDIS_DATA_LOCATION}'
      db-data:
        device: !ENV '${DB_DATA_LOCATION}'
    ```

    **immich-postgres-template.conf**:

    ```
    exec.start: "/bin/sh /etc/rc"
    exec.stop: "/bin/sh /etc/rc.shutdown jail"
    sysvmsg: new
    sysvsem: new
    sysvshm: new
    mount.devfs
    persist
    ```

    **Makejail**:

    ```
    OPTION container=boot args:--pull
    OPTION overwrite=force
    ```

Access Immich at: **http://your-host:2283**

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `UPLOAD_LOCATION` | `/containers/immich/library` | Path to store uploaded photos and videos |
| `DB_DATA_LOCATION` | `/containers/immich/postgres` | Path to store PostgreSQL database files |
| `DB_PASSWORD` | `postgres` | PostgreSQL password (**change this!**) |
| `DB_USERNAME` | `postgres` | PostgreSQL username |
| `DB_DATABASE_NAME` | `immich` | PostgreSQL database name |
| `TZ` | System default | Timezone (e.g., `America/Los_Angeles`) |

## Ports

| Port | Service | Description |
|------|---------|-------------|
| `2283` | immich-server | Web UI and API |
| `3003` | immich-ml | Machine Learning API (internal) |
| `5432` | immich-postgres | PostgreSQL (internal) |
| `6379` | redis | Redis cache (internal) |

!!! note "Network Mode"
    The stack uses `network_mode: host` for simplicity on FreeBSD. All services communicate via localhost. Only port 2283 needs to be exposed externally.

## FreeBSD-Specific Notes

### PostgreSQL Shared Memory

PostgreSQL requires System V IPC for shared memory. The compose file includes:

```yaml
annotations:
  org.freebsd.jail.allow.sysvipc: "true"
```

This annotation is processed by ocijail to allow the jail access to shared memory.

### Machine Learning Performance

The `immich-ml` container uses:

- **Native FreeBSD onnxruntime** - Custom-built wheel with FreeBSD support
- **CPU inference** - GPU acceleration not currently supported on FreeBSD
- **HuggingFace models** - Downloaded on first run (~1GB cache)

First startup may be slow as ML models are downloaded. Subsequent starts are fast.

## Management

### View Logs

```bash
# All services
podman-compose logs -f

# Specific service
podman logs -f immich_server
podman logs -f immich_machine_learning
```

### Restart Stack

```bash
podman-compose restart
```

### Stop Stack

```bash
podman-compose down
```

### Update Images

```bash
podman-compose pull
podman-compose up -d
```

## Component Documentation

For detailed configuration of individual services:

| Component | Documentation |
|-----------|---------------|
| :material-server: **Server** | [immich-server](immich-server.md) |
| :material-brain: **Machine Learning** | [immich-ml](immich-ml.md) |
| :simple-postgresql: **PostgreSQL** | [immich-postgres](immich-postgres.md) |
| :simple-redis: **Redis** | [redis](redis.md) |

## Troubleshooting

??? question "ML service keeps restarting"
    Check memory - the ML service needs at least 2GB RAM:
    ```bash
    podman logs immich_machine_learning
    ```
    If you see OOM errors, increase available memory or disable ML features in Immich settings.

??? question "Database connection errors"
    Ensure PostgreSQL has started and the sysvipc annotation is working:
    ```bash
    podman logs immich_postgres
    ```
    The database needs a few seconds to initialize on first run.

??? question "Photos not uploading from mobile"
    Ensure port 2283 is accessible from your network. Check firewall rules:
    ```bash
    # PF firewall
    pass in on egress proto tcp to port 2283
    ```

??? question "Facial recognition not working"
    The ML service downloads models on first use. Check if models are cached:
    ```bash
    podman exec immich_machine_learning ls -la /cache/huggingface
    ```

!!! info "Implementation Details"

    - **Platform:** Native FreeBSD 15.0 (no Linux emulation)
    - **Runtime:** ocijail (FreeBSD jails as OCI containers)
    - **ML Backend:** onnxruntime with custom FreeBSD wheel
    - **Database:** PostgreSQL 14 with pgvector extension
    - **Process Manager:** s6-overlay

[Immich Website](https://immich.app/){ .md-button .md-button--primary }
[Immich Docs](https://immich.app/docs/){ .md-button }
[Source Code](https://github.com/immich-app/immich){ .md-button }
