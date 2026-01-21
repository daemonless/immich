# Immich for FreeBSD (Daemonless)

Self-hosted photo and video management solution.

| | |
|---|---|
| **Docs** | [daemonless.io/images/immich](https://daemonless.io/images/immich/) |
| **Website** | [immich.app](https://immich.app/) |

## Deployment Instructions

```sh
mkdir -p /containers/immich
cd /containers/immich
fetch https://raw.githubusercontent.com/daemonless/immich/main/container-compose.yml -o container-compose.yml
fetch https://raw.githubusercontent.com/daemonless/immich/main/example.env -o .env
```

### Edit .env file:

```sh
vi .env  # Set UPLOAD_LOCATION and DB_PASSWORD
```

### Create library directory with required subdirectories:

```sh
mkdir -p /containers/immich/library
for dir in thumbs upload backups library profile encoded-video; do
  mkdir -p /containers/immich/library/$dir
  touch /containers/immich/library/$dir/.immich
done
chown -R 1000:1000 /containers/immich/library
```