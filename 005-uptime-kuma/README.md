# Uptime Kuma Docker Compose setup: monitor your homelab with push notifications (ntfy)

▶ [Watch the video](https://youtu.be/pExPTlWe5OU) · EP 005 · Tested on: Debian 13 (trixie), Uptime Kuma 2.5.11, Caddy 2.11.7, Vaultwarden 1.37.5, Gitea 28.2.0

Uptime Kuma setup with Docker Compose: self-hosted monitoring for your homelab, with push notifications through ntfy. We put it behind the same Caddy reverse proxy as Gitea and Vaultwarden, with HTTPS on a local name, add HTTP monitors for both services, and then stop one of them to watch the alert arrive.

## Intro

[▶ 0:00](https://youtu.be/pExPTlWe5OU?t=0s)

**What you need**

- Caddy, Gitea and Vaultwarden from **episodes 1–4**
- A name for the dashboard, like `uptime.home.arpa`
- Optional: the free **ntfy** app on your phone

## Local name for the dashboard

[▶ 0:27](https://youtu.be/pExPTlWe5OU?t=27s)

On the workstation, add one more name: uptime.home.arpa, pointing at the same server.

*On your workstation:*

```bash
echo "SERVER_IP uptime.home.arpa" | sudo tee -a /etc/hosts
grep home.arpa /etc/hosts
```

## Uptime Kuma in Docker Compose

[▶ 0:40](https://youtu.be/pExPTlWe5OU?t=40s)

On the server, Uptime Kuma gets its own directory, like Vaultwarden. Copy Caddy's root certificate into it. You'll see why in a moment. The compose file. One service, pinned to version 2.5.11. It joins the proxy network, with no published ports, so Caddy is the only way in. extra_hosts works like the hosts file on your workstation. Inside the container, the Git and vault names point at the server itself. So Kuma checks the same HTTPS addresses you use, through Caddy. A broken certificate or proxy also counts as down. Those certificates come from Caddy's own authority, so Node has to trust its root. That's NODE_EXTRA_CA_CERTS. Without it, every check fails with a certificate error.

Create `~/uptime-kuma/compose.yaml` ([download](compose.yaml)):

```yaml
services:
  uptime-kuma:
    image: docker.io/louislam/uptime-kuma:2.5.11
    restart: always
    environment:
      - NODE_EXTRA_CA_CERTS=/certs/caddy-root.crt
    extra_hosts:
      - "git.home.arpa:host-gateway"
      - "vault.home.arpa:host-gateway"
    volumes:
      - ./data:/app/data
      - ./caddy-root.crt:/certs/caddy-root.crt:ro
    networks:
      - proxy

networks:
  proxy:
    external: true
```

*On the server:*

```bash
cp ../gitea/caddy-root.crt .
```

## Third site in Caddy

[▶ 1:24](https://youtu.be/pExPTlWe5OU?t=84s)

In the Caddyfile, a third site block. Uptime Kuma listens on port 3001. This time, the compose file stays the same, so compose has nothing to recreate. Instead, tell Caddy to reload its config. The new site goes live without dropping any connections.

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
```

*On the server:*

```bash
tail -4 Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

> **Edited the Caddyfile in an editor and the reload changed nothing?** The `Caddyfile` is bind-mounted as a single file.
> Some editors save by writing a new file and renaming it, and the container keeps seeing the old one.
> Run `docker compose restart caddy` in that case.

Start Uptime Kuma. Five containers now, and still only Caddy publishes web ports.

```bash
docker compose up -d
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

## First start

[▶ 1:57](https://youtu.be/pExPTlWe5OU?t=117s)

Open uptime.home.arpa. On first start, Kuma asks for a database. SQLite is plenty for a home lab: a single file in the data directory. Then create the admin account. Your new password manager comes in handy here.

## Monitors and notifications

[▶ 2:17](https://youtu.be/pExPTlWe5OU?t=137s)

Add a monitor. The HTTP type requests a page, and expects a good answer. Name it Gitea, and enter the same address you use in the browser. Kuma checks every sixty seconds. With retries at zero, one failed check is enough to raise an alert.

Now the alert. Kuma supports dozens of services, like email, Telegram or Discord. We'll use ntfy. It's open source, and needs no account. On the public ntfy server, anyone who knows a topic name can read it, so make it hard to guess. Or host ntfy yourself. Enable it by default, so every new monitor uses it. Send a test, save, and save the monitor.

In the ntfy app, or on its web page, subscribe to the same topic. The test message is already there.

> **ntfy on your phone:**
>
> 1. Install the ntfy app (Android: Google Play or F-Droid, iOS: App Store).
> 2. Tap **+**, enter the same topic name as in Uptime Kuma, and keep the server `ntfy.sh`.
>    If you host ntfy yourself, turn on "Use another server" and enter its address.
> 3. On Android, allow the app to run in the background (no battery optimisation), or alerts may arrive late.
>
> In a browser, open `https://ntfy.sh/your-topic`, or use **Subscribe to topic** in the ntfy web app.

One more monitor, for Vaultwarden. It has a health check at /alive. The notification is already ticked, because it's on by default. Both services are up. Each bar is one check.

## Break something

[▶ 3:54](https://youtu.be/pExPTlWe5OU?t=234s)

Now, let's break something. Stop Vaultwarden.

*On the server:*

```bash
docker compose stop
```

Within a minute, Kuma marks it down, with status code 502. Caddy still answers, but nothing is behind it. And the alert is on your phone. It tells you which service failed, and why.

Start it again.

```bash
docker compose start
```

Kuma sees it come back, and sends one more message: it's up. It also tracks uptime and response times, and warns you before a certificate expires.

## Next steps

- Add a monitor for every service you run
- Share a **status page** with your users
- Next episode: **restic** backups

## Links

- https://github.com/louislam/uptime-kuma/wiki
- https://docs.ntfy.sh/
- https://docs.docker.com/reference/compose-file/services/#extra_hosts

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
