# Self-host Gitea with Docker Compose and PostgreSQL

▶ [Watch the video](https://youtu.be/znjTBW9-Us4) · EP 001 · Tested on: Debian 13 (trixie), Gitea 28.2.0, PostgreSQL 18

Self-host Gitea 28.2.0 with Docker Compose and PostgreSQL 18 on Debian 13 (trixie), then push your first repository over SSH.

## Intro

[▶ 0:00](https://youtu.be/znjTBW9-Us4?t=0s)

**What you need**

- A Linux server: **Debian 13**, 2 vCPU, 2 GB RAM
- A user with `sudo`
- Ports `3000` (web) and `2222` (SSH) reachable

## Install Docker

[▶ 0:29](https://youtu.be/znjTBW9-Us4?t=29s)

Let's connect to the server over SSH. A quick check confirms Debian 13, two cores and two gigabytes of memory.

*On your workstation:*

```bash
ssh YOUR_USER@SERVER_IP
grep PRETTY_NAME /etc/os-release
nproc && free -h
```

We'll install Docker from its official repository, not from the Debian packages, to get the current version with the Compose plugin. First, download Docker's signing key into the apt keyrings directory. Then add the repository for our Debian release.

*On the server:*

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

Now install the Docker engine, the command line client and the Compose plugin. Add your user to the docker group, so you don't need sudo for every command, and start a new shell with that group. Docker and Docker Compose are ready.

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker --version && docker compose version
```

## Compose file

[▶ 1:16](https://youtu.be/znjTBW9-Us4?t=76s)

Create a directory for the project. Instead of a hard-coded database password, generate a random one and keep it in a .env file. Add the address your users will reach Gitea at. I'm using the server's IP, but a domain name works too. Then make the file readable only by your user.

*On the server:*

```bash
mkdir ~/gitea && cd ~/gitea
echo "POSTGRES_PASSWORD=$(openssl rand -hex 16)" > .env
echo GITEA_DOMAIN=SERVER_IP >> .env
chmod 600 .env && ls -l .env
```

The server service runs Gitea 28.2, pinned to an exact version, so an update never happens by surprise. The database settings are passed as environment variables, and the password comes from the .env file. The next four lines matter: they set the domain, the SSH port and the base URL. The Gitea 28 installer no longer asks for the domain, so without them your clone addresses would point to localhost. Port 3000 serves the web interface, and port 2222 on the host maps to SSH inside the container. One gotcha on Debian 13: the official example also mounts /etc/timezone, but that file doesn't exist here, and Docker would silently create an empty directory instead. So we only mount localtime. The db service runs PostgreSQL 18. Note the volume path: since version 18, the data lives under /var/lib/postgresql.

Create `~/gitea/compose.yaml` ([download](compose.yaml)):

```yaml
services:
  server:
    image: docker.gitea.com/gitea:28.2.0
    restart: always
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=${POSTGRES_PASSWORD}
      - GITEA__server__DOMAIN=${GITEA_DOMAIN}
      - GITEA__server__SSH_DOMAIN=${GITEA_DOMAIN}
      - GITEA__server__SSH_PORT=2222
      - GITEA__server__ROOT_URL=http://${GITEA_DOMAIN}:3000/
    volumes:
      - ./gitea:/data
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
    depends_on:
      - db

  db:
    image: docker.io/library/postgres:18
    restart: always
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=gitea
    volumes:
      - ./postgres:/var/lib/postgresql
```

Start everything with docker compose up, in detached mode. Docker pulls both images and starts the containers. Both containers are up, and Gitea answers on port 3000.

```bash
docker compose up -d
docker compose ps --format "table {{.Service}}\t{{.Image}}\t{{.Status}}"
curl -sI http://localhost:3000 | head -1
```

## Web installer

[▶ 2:44](https://youtu.be/znjTBW9-Us4?t=164s)

Open port 3000 in your browser, and you'll see the initial configuration page. The database section is already filled in from the environment variables, so leave it as it is. Give your instance a name. The SSH port and the base URL are already set from the compose file, so there's nothing to change here. Finally, expand the administrator account settings and create the first admin user. Click install, and after a few seconds you're logged in to your own Gitea.

## First repository

[▶ 3:43](https://youtu.be/znjTBW9-Us4?t=223s)

Let's create a repository. Click the plus icon and choose new repository. Call it hello-gitea, and create it. Gitea shows the commands to push an existing project. Switch to SSH, and you get the address with port 2222.

To push over SSH, your workstation needs a key. If you already have one, skip this step. Generate an ed25519 key. For a real machine, protect it with a passphrase. Then print the public key and copy it.

*On your workstation:*

```bash
ssh-keygen -t ed25519 -C "you@workstation"
cat ~/.ssh/id_ed25519.pub
```

In Gitea, open settings, then SSH and GPG keys, and click add key. Paste the public key, give it a name, and save it.

Back on the workstation, create a small project and make the first commit. Add Gitea as the remote, using the SSH address from the repository page. Push. The first time, SSH asks you to confirm the server's host key. Type yes. And the push goes through.

```bash
mkdir hello-gitea && cd hello-gitea
git init -b main
echo "# Hello Gitea" > README.md
git add README.md
git commit -m "First commit"
git remote add origin ssh://git@SERVER_IP:2222/labadmin/hello-gitea.git
git push -u origin main
```

Refresh the repository page, and there it is: your README and your first commit, on a Git server you fully control.

## Next steps

- Put Gitea behind a **reverse proxy** with HTTPS
- Back up `./gitea` and `./postgres`
- Update by changing the image tag, then `docker compose up -d`

## Links

- https://docs.gitea.com/installation/install-with-docker

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
