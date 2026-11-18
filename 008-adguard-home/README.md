# AdGuard Home Docker Compose setup: block ads on your whole network + local DNS for your homelab

▶ [Watch the video](https://youtu.be/KQttRigyk0c) · EP 008 · Tested on: Debian 13 (trixie), AdGuard Home 0.107.80, Caddy 2.11.7, Docker Compose

AdGuard Home with Docker Compose on a Debian 13 homelab server: fix the 'address already in use' error on port 53 (systemd-resolved), run the setup wizard behind Caddy, add one DNS rewrite so every *.home.arpa name points to the server, and block ads and trackers for every device on the network. No more /etc/hosts edits.

## Port 53 is taken

[▶ 0:19](https://youtu.be/KQttRigyk0c?t=19s)

The compose file. AdGuard Home answers DNS on port 53, over UDP and TCP. Its web panel stays inside, on the proxy network, behind Caddy. Start it, and here's the error almost everyone hits: address already in use. Something already listens on port 53. It's systemd-resolved, the local DNS cache that Debian and Ubuntu run by default.

Create `~/adguard/compose.yaml` ([download](compose.yaml)):

```yaml
services:
  adguardhome:
    image: docker.io/adguard/adguardhome:v0.107.80
    restart: always
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

Create `~/adguard/resolved.conf` ([download](resolved.conf)):

```
# AdGuard Home needs port 53: turn off the local DNS stub of systemd-resolved
[Resolve]
DNSStubListener=no
```

*On the server:*

```bash
docker compose up -d
sudo ss -lntup 'sport = :53'
```

One setting turns off its stub listener, and frees the port. Install it as a drop-in file. Then point resolv.conf at the real upstream servers, instead of the stub that's now gone. Restart resolved, and start AdGuard Home again. This time, the container starts.

```bash
cat resolved.conf
sudo install -D -m 644 resolved.conf /etc/systemd/resolved.conf.d/adguardhome.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
docker compose up -d
```

## Setup

[▶ 1:16](https://youtu.be/KQttRigyk0c?t=76s)

In Caddy, one more site: adguard.home.arpa, to the panel on port 3000. Reload Caddy.

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
```

*On the server:*

```bash
tail -4 Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

And on the workstation, one hosts entry, for the last time.

*On your workstation:*

```bash
echo 'SERVER_IP adguard.home.arpa' | sudo tee -a /etc/hosts
```

The setup wizard. Keep the admin panel on port 3000, because that's where Caddy sends the traffic. DNS stays on port 53. Create the admin account, and skip the device instructions for now. Then log in through Caddy, with HTTPS. The default block list is already on, with a few hundred thousand rules.

## One rule for every name

[▶ 2:15](https://youtu.be/KQttRigyk0c?t=135s)

Now the part that replaces every hosts file. Filters, then DNS rewrites. *.home.arpa, pointing at the server. Every name, today's and future ones, in one rule.

## Use it

[▶ 2:33](https://youtu.be/KQttRigyk0c?t=153s)

Back on the workstation, delete the old entries from the hosts file. Every single one. Then use AdGuard Home as the DNS server. Here, just for this laptop. For the whole house, you set it once in your router. The Git server's name now comes from DNS. And an advertising server gets nowhere: 0.0.0.0.

*On your workstation:*

```bash
sudo sed -i '/home.arpa/d' /etc/hosts
sudo resolvectl dns eth0 SERVER_IP
resolvectl query git.home.arpa
resolvectl query doubleclick.net
```

**For the whole house**

- Router, DHCP settings: DNS server = **your server's IP**
- Don't add a second DNS: devices would skip the filter
- Keep the server up: no DNS, no internet

In the browser, the Git server opens by name, with no hosts entry. And the query log shows every request from every device. The red ones are ads and trackers that never loaded.

## Next steps

- Router DHCP: **one DNS**, your server
- Add a DNS monitor in Uptime Kuma
- Next episode: **Immich**, your own photo library

## Links

- https://github.com/AdguardTeam/AdGuardHome/wiki/Docker
- https://github.com/AdguardTeam/AdGuardHome/wiki/FAQ#bindinuse
- https://adguard-dns.io/kb/general/dns-filtering-syntax/#dnsrewrite-modifier

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
