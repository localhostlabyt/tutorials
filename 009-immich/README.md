# Immich Docker Compose setup: your own Google Photos alternative (Caddy HTTPS + nightly backups)

▶ [Watch the video](https://youtu.be/uIog9BlkCco) · EP 009 · Tested on: Debian 13 (trixie), Immich 3.4.1, Caddy 2.11.7, AdGuard Home 0.107.80, restic 0.18, Docker Compose

Immich with Docker Compose on a Debian 13 homelab server: the official compose and env files for v3.4.1 with two changes, HTTPS through Caddy on photos.home.arpa (the name comes from AdGuard Home, no hosts file), the first admin account, uploads with timeline, map and smart search, the phone app, and adding Immich to the nightly restic backup from episode 6.

## Install

[▶ 0:24](https://youtu.be/uIog9BlkCco?t=24s)

Immich publishes a compose file and an env file with every release. Download both, for version 3.4.1. Next to them is my version from the guide. It changes only two things. No published port. Instead, the server joins the proxy network, and Caddy handles HTTPS, like for every other app. Then remove the original, so only one compose file is left.

Create `~/immich/compose.yaml` ([download](compose.yaml)):

```yaml
#
# WARNING: To install Immich, follow our guide: https://docs.immich.app/install/docker-compose
#
# Make sure to use the docker-compose.yml of the current release:
#
# https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
#
# The compose file on main may not be compatible with the latest release.

name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    # extends:
    #   file: hwaccel.transcoding.yml
    #   service: cpu # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
    volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      - ${UPLOAD_LOCATION}:/data
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    networks:
      - default
      - proxy
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    # extends: # uncomment this section for hardware acceleration - see https://docs.immich.app/features/ml-hardware-acceleration
    #   file: hwaccel.ml.yml
    #   service: cpu # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    environment:
      MACHINE_LEARNING_MODEL_REVISION: v2
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9@sha256:418652cfb58ef879d4978c33553735d7147016032d5aefaa14c828e611eb9dfd
    healthcheck:
      test: redis-cli ping | grep -q PONG || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # Uncomment the DB_STORAGE_TYPE: 'HDD' var if your database isn't stored on SSDs
      # DB_STORAGE_TYPE: 'HDD'
    volumes:
      # Do not edit the next line. If you want to change the database storage location on your system, edit the value of DB_DATA_LOCATION in the .env file
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    healthcheck:
      disable: false

volumes:
  model-cache:

networks:
  proxy:
    external: true
```

*On the server:*

```bash
curl -sLO https://github.com/immich-app/immich/releases/download/v3.4.1/docker-compose.yml
curl -sL -o .env https://github.com/immich-app/immich/releases/download/v3.4.1/example.env
ls -A
diff docker-compose.yml compose.yaml
rm docker-compose.yml
```

In the env file, set a random database password. Letters and digits only. And pin the version, so updates happen when you decide. Photos go to the library folder, and the database to the postgres folder. Both live right here, in the home directory.

```bash
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 16)/" .env
sed -i 's/^IMMICH_VERSION=.*/IMMICH_VERSION=v3.4.1/' .env
grep -v '^#' .env | grep .
```

Start it. Four containers: the server, machine learning for search and faces, Valkey as the cache, and Postgres. The server needs about a minute for its first start. Immich asks for at least six gigabytes of memory, mostly for machine learning. With four, turn machine learning off.

```bash
docker compose up -d
docker compose ps --format "table {{.Service}}\t{{.Status}}"
```

## HTTPS and a name

[▶ 1:30](https://youtu.be/uIog9BlkCco?t=90s)

In Caddy, one more site: photos.home.arpa. It points to Immich, on port 2283. Reload Caddy.

Create `~/gitea/Caddyfile` ([download](Caddyfile)):

```
{$GITEA_DOMAIN} {
	tls internal
	reverse_proxy server:3000
}

vault.home.arpa {
	tls internal
	reverse_proxy vaultwarden:80
}

uptime.home.arpa {
	tls internal
	reverse_proxy uptime-kuma:3001
}

adguard.home.arpa {
	tls internal
	reverse_proxy adguardhome:3000
}

photos.home.arpa {
	tls internal
	reverse_proxy immich-server:2283
}
```

*On the server:*

```bash
tail -4 Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

And no hosts file this time. The AdGuard rule from episode eight already answers for the new name.

*On your workstation:*

```bash
resolvectl query photos.home.arpa
```

## First login

[▶ 1:48](https://youtu.be/uIog9BlkCco?t=108s)

Open it in the browser. The first account you create becomes the admin. Then a short setup: theme, privacy, and the folder layout for your files. Immich even reminds you about backups. We will get to that at the end.

## Your photos

[▶ 2:44](https://youtu.be/uIog9BlkCco?t=164s)

Upload a few photos. Normally the phone app does this for you, in the background. Immich reads the date and the place from every photo, and builds the timeline. The original files stay untouched, in the library folder on the server.

The map puts every photo where it was taken. And search understands what's in the picture. Bridge at night finds exactly that.

**On your phone**

- Immich app: **Android & iOS**
- Server: **https://photos.home.arpa**
- Backup → camera album → **background backup**
- Away from home: **WireGuard** from episode 7

## Backups

[▶ 3:35](https://youtu.be/uIog9BlkCco?t=215s)

Photos are the one thing you can't download again. So Immich joins the nightly backup from episode six. As root, add its folder to the list of apps in the backup script. Run it once by hand. Immich stops with the others, restic copies the photos and the database, and everything starts again. Immich also writes its own database dump every night at two, into the library folder. The backup at three copies that file too. And the new snapshot is on the NAS.

*On the server, as root (`sudo -i`):*

```bash
sed -i 's#^APPS="#APPS="/home/YOUR_USER/immich #' /usr/local/sbin/homelab-backup
grep APPS= /usr/local/sbin/homelab-backup
homelab-backup
. /etc/restic/env && restic snapshots
```

## Next steps

- Phone app: turn on **background backup**
- Library on a disk with **room to grow**
- Next episode: **Nextcloud**, files and calendars

## Links

- https://docs.immich.app/install/docker-compose
- https://docs.immich.app/install/requirements
- https://docs.immich.app/administration/backup-and-restore

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
