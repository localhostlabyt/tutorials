# Vaultwarden setup with Docker Compose: self-hosted Bitwarden password manager (HTTPS with Caddy)

▶ [Watch the video](https://youtu.be/phirbPfpAoA) · EP 004 · Tested on: Debian 13 (trixie), Vaultwarden 1.37.5, Caddy 2.11.7, Gitea 28.2.0

Vaultwarden setup with Docker Compose: a lightweight, Bitwarden-compatible password manager on your own server. We put it behind the same Caddy reverse proxy as Gitea from episode 3, with HTTPS on a local name, create the first account, save a login with the password generator, and then disable public sign-ups.

## Intro

[▶ 0:00](https://youtu.be/phirbPfpAoA?t=0s)

**What you need**

- Caddy with HTTPS from **episode 3**
- Its root certificate trusted on your devices
- A name for the vault, like `vault.home.arpa`

## Local name for the vault

[▶ 0:34](https://youtu.be/phirbPfpAoA?t=34s)

On the workstation, add one more name: vault.home.arpa, pointing at the same server, next to the Git name from last time. Both names lead to the same machine. Caddy decides which app gets each request.

*On your workstation:*

```bash
echo "SERVER_IP vault.home.arpa" | sudo tee -a /etc/hosts
grep home.arpa /etc/hosts
```

## Shared Docker network

[▶ 0:47](https://youtu.be/phirbPfpAoA?t=47s)

On the server, Vaultwarden gets its own directory and its own compose project. But Caddy lives in the Gitea project. Each compose project has a private network, so the two can't see each other yet. A shared network fixes that. Call it proxy. We'll change two files in the Gitea directory, so back them up first.

*On the server:*

```bash
docker network create proxy
cp Caddyfile Caddyfile.bak
cp compose.yaml compose.yaml.bak
```

## Vaultwarden in Docker Compose

[▶ 1:06](https://youtu.be/phirbPfpAoA?t=66s)

Next to it, a vaultwarden directory with this compose file. One service, pinned to version 1.37.5. DOMAIN is the public address of the vault, with HTTPS. Sign-ups stay open for now, so we can create the first account. Everything Vaultwarden stores goes into the data directory. There are no ports at all. The container only joins the proxy network, so Caddy is the only way in.

Create `~/vaultwarden/compose.yaml` ([download](vaultwarden-compose.yaml)):

```yaml
services:
  vaultwarden:
    image: docker.io/vaultwarden/server:1.37.5
    restart: always
    environment:
      - DOMAIN=https://vault.home.arpa
      - SIGNUPS_ALLOWED=true
    volumes:
      - ./data:/data
    networks:
      - proxy

networks:
  proxy:
    external: true
```

## Second site in Caddy

[▶ 1:32](https://youtu.be/phirbPfpAoA?t=92s)

In the Caddyfile, a second site block. Same internal certificate authority, and requests go to the vaultwarden container, on port 80. Inside the network, that's plain HTTP, but it never leaves the server. In the Gitea compose file, Caddy joins the proxy network, and keeps the default one, so it still reaches Gitea. external: true means compose uses the network we created, instead of making its own.

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
```

Create `~/gitea/compose.yaml` ([download](gitea-compose.yaml)):

```yaml
services:
  caddy:
    image: docker.io/library/caddy:2.11.7
    restart: always
    environment:
      - GITEA_DOMAIN=${GITEA_DOMAIN}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    networks:
      - default
      - proxy

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
      - GITEA__server__ROOT_URL=https://${GITEA_DOMAIN}/
    volumes:
      - ./gitea:/data
      - /etc/localtime:/etc/localtime:ro
    ports:
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

networks:
  proxy:
    external: true
```

*On the server:*

```bash
diff --color -U1 Caddyfile.bak Caddyfile
diff --color -U1 compose.yaml.bak compose.yaml
```

Start Vaultwarden. Then apply the Gitea project again. Compose recreates only Caddy, with the new network and the new site. Four containers. Only Caddy serves the web, and Vaultwarden has no published ports at all.

```bash
docker compose up -d
cd ../gitea && docker compose up -d
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

## Create your account

[▶ 2:18](https://youtu.be/phirbPfpAoA?t=138s)

Open vault.home.arpa. It's a valid HTTPS page right away, because the browser already trusts Caddy's authority. Next, create an account. Without an email server, Vaultwarden skips the verification email. Now the master password. It encrypts the whole vault, and the server never sees it. If you forget it, nobody can recover your data, not even the admin. Skip the extension prompt for now. We'll come back to the apps at the end.

Let's save a first login: the Gitea account from episode one. Instead of typing a password, use the generator. Long and random, and you never have to remember it. You can pick the length and the character sets, or switch to a passphrase made of random words. Add the website, so the apps can offer to fill it in, and save.

## Lock down sign-ups

[▶ 3:22](https://youtu.be/phirbPfpAoA?t=202s)

Your account exists, so close the door behind you. Set SIGNUPS_ALLOWED to false, and apply. The data directory is everything you need to back up: the database, and the key that signs logins. The admin panel stays off, because we never set an admin token.

*On the server:*

```bash
sed -i 's/SIGNUPS_ALLOWED=true/SIGNUPS_ALLOWED=false/' compose.yaml
grep SIGNUPS compose.yaml
docker compose up -d
ls data
```

A stranger who finds the page can no longer create an account. Vaultwarden answers: registration not allowed. To add your family later, open sign-ups for a minute, or set up email and send invitations.

## Apps and browser extension

[▶ 4:09](https://youtu.be/phirbPfpAoA?t=249s)

**Bitwarden apps**

- On the login screen choose **self-hosted**
- Server URL: `https://vault.home.arpa`
- The device must trust Caddy's root certificate

## Next steps

- Back up `./data` regularly
- Keep sign-ups off, invite people instead
- Next episode: **Uptime Kuma** monitoring

## Links

- https://github.com/dani-garcia/vaultwarden/wiki
- https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples
- https://docs.docker.com/compose/how-tos/networking/

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
