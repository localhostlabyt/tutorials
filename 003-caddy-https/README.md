# Caddy reverse proxy with automatic HTTPS for self-hosted apps (Gitea + Docker Compose)

▶ [Watch the video](https://youtu.be/YQueGn2O6bM) · EP 003 · Tested on: Debian 13 (trixie), Caddy 2.11.7, Gitea 28.2.0, PostgreSQL 18

Caddy reverse proxy setup with Docker Compose: put Caddy 2.11 in front of Gitea from episode 1 and get HTTPS on your local network with tls internal, an HTTP to HTTPS redirect and port 3000 closed. Plus the one change you need for Let's Encrypt with a real domain.

## Intro

[▶ 0:00](https://youtu.be/YQueGn2O6bM?t=0s)

**What you need**

- Gitea in Docker Compose from **episode 1**
- Ports `80` and `443` free on the server
- A domain, or a local name like `git.home.arpa`

## Why HTTPS

[▶ 0:34](https://youtu.be/YQueGn2O6bM?t=34s)

This is Gitea today. The browser marks the page as not secure. And for a good reason. When you sign in, your password travels over the network in plain text.

## Local domain (home.arpa)

[▶ 0:47](https://youtu.be/YQueGn2O6bM?t=47s)

A certificate is issued for a name, so the server needs one. With a real domain, you'd create a DNS record that points to the server. My lab has no public domain, so I use git.home.arpa. The home.arpa domain is reserved for home networks, so it never clashes with a real website. On your workstation, add it to the hosts file. Or add it to your router's DNS, so every device on your network knows it.

*On your workstation:*

```bash
echo "SERVER_IP git.home.arpa" | sudo tee -a /etc/hosts
getent hosts git.home.arpa
```

## Caddyfile with tls internal

[▶ 1:14](https://youtu.be/YQueGn2O6bM?t=74s)

On the server, in the gitea directory, first back up the compose file. Then create a file named Caddyfile. It's just four lines. The site address comes from the same variable that Gitea uses. reverse_proxy sends every request to the Gitea container, on port 3000. tls internal tells Caddy to sign the certificate with its own local authority, because Let's Encrypt can't reach a name that only exists in your home network. Finally, switch the domain in the .env file to the new name.

Create `~/gitea/Caddyfile` ([download](Caddyfile)):

```
{$GITEA_DOMAIN} {
	tls internal
	reverse_proxy server:3000
}
```

*On the server:*

```bash
cp compose.yaml compose.yaml.bak
sed -i 's/^GITEA_DOMAIN=.*/GITEA_DOMAIN=git.home.arpa/' .env
grep DOMAIN .env
```

## Caddy in Docker Compose

[▶ 1:45](https://youtu.be/YQueGn2O6bM?t=105s)

Now the compose file. Here are the changes, compared to the backup. A new caddy service, pinned to version 2.11.7. It gets the domain, the Caddyfile, and two data directories. One of them keeps the certificates. Caddy listens on three ports. Port 80 only redirects to HTTPS. Port 443 serves HTTPS. And the same port over UDP serves HTTP/3. Gitea's base URL now starts with HTTPS, and has no port number. And port 3000 is no longer published. Gitea can only be reached through Caddy. SSH on port 2222 stays as it was.

Create `~/gitea/compose.yaml` ([download](compose.yaml)):

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
```

*On the server:*

```bash
diff --color -U1 compose.yaml.bak compose.yaml
```

Apply the changes with docker compose up. Compose recreates Gitea with the new settings, and starts Caddy, which issues the certificate right away. All three containers are running. Copy Caddy's root certificate out of the container. Every device that should trust this local authority needs a copy of this file.

```bash
docker compose up -d
docker compose ps --format "table {{.Service}}\t{{.Image}}\t{{.Status}}"
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt caddy-root.crt
```

## Trust Caddy's local CA

[▶ 2:47](https://youtu.be/YQueGn2O6bM?t=167s)

Back on the workstation, curl doesn't trust the certificate yet. It was signed by an authority it has never seen. Copy the root certificate from the server. Add it to the system's trusted certificates, and update the store. Now curl gets an answer over HTTP/2, with a valid certificate.

*On your workstation:*

```bash
curl -I https://git.home.arpa
scp YOUR_USER@SERVER_IP:gitea/caddy-root.crt .
sudo cp caddy-root.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
curl -I https://git.home.arpa
```

Chrome and Chromium on Linux keep their own certificate database. certutil, from the libnss3-tools package, adds the root certificate there. On Windows and macOS, open the file and add it to the trusted root certificates. Firefox has its own import button, under Privacy & Security, Certificates. Then restart the browser.

```bash
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Caddy lab CA" -i caddy-root.crt
certutil -d sql:$HOME/.pki/nssdb -L
```

## HTTPS works

[▶ 3:40](https://youtu.be/YQueGn2O6bM?t=220s)

Open git.home.arpa. The "Not secure" warning in the address bar is gone. Sign in. This time, your password is encrypted on its way to the server. And Gitea's clone address now uses HTTPS too.

A few checks from the terminal. Plain HTTP gets a permanent redirect to HTTPS. Caddy does that on its own. Port 3000 is closed, so nobody can go around the proxy. And git clone over HTTPS just works, because the system trusts the certificate.

*On your workstation:*

```bash
curl -sI http://git.home.arpa | head -3
curl http://SERVER_IP:3000
git clone https://git.home.arpa/labadmin/hello-gitea.git
```

## Let's Encrypt with a real domain

[▶ 4:18](https://youtu.be/YQueGn2O6bM?t=258s)

**With a real domain**

- Point a DNS **A record** at your server
- Delete the `tls internal` line
- Open ports `80` and `443` to the internet

## Next steps

- Back up `./caddy/data`: it holds your local CA key
- More services? One site block per name
- Next episode: **Vaultwarden** behind Caddy

## Links

- https://caddyserver.com/docs/quick-starts/reverse-proxy
- https://caddyserver.com/docs/caddyfile/directives/tls
- https://www.rfc-editor.org/rfc/rfc8375

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
