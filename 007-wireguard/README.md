# WireGuard VPN setup on Debian 13: reach your homelab from anywhere (laptop + phone QR code)

▶ [Watch the video](https://youtu.be/tEgerx0zY3g) · EP 007 · Tested on: Debian 13 (trixie), wireguard-tools 1.0.20210914, Gitea 28.2.0, Vaultwarden 1.37.5

WireGuard VPN setup on a Debian 13 homelab server: one UDP port, a key pair for every device, and a small script that adds clients. We connect a Linux laptop with wg-quick, open the self-hosted apps (Vaultwarden, Gitea) through the encrypted tunnel, and add a phone by scanning a QR code.

## WireGuard server

[▶ 0:16](https://youtu.be/tEgerx0zY3g?t=16s)

WireGuard is a VPN built into the Linux kernel. Every device gets its own key pair, and the server listens on one UDP port. On the server, install it, and qrencode for the phone.

*On the server:*

```bash
sudo apt install -y wireguard qrencode
wg --version
```

As root, in /etc/wireguard. umask 077 makes every new file readable only by root, keys included. Generate the server's key pair. The server config: its address in the tunnel, 10.8.0.1, the port, and its private key.

*On the server, as root (`sudo -i`):*

```bash
umask 077
wg genkey | tee server.key | wg pubkey > server.pub
cat > wg0.conf <<EOF
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = $(cat server.key)
EOF
ls
```

## Clients

[▶ 0:51](https://youtu.be/tEgerx0zY3g?t=51s)

Every new device needs the same three steps, so here's a small script. It creates a key pair for the device. It adds the device to the server config, with its public key and its address in the tunnel. And it writes the device's own config: its private key, the server's public key, and where to find the server. AllowedIPs sends only the tunnel network through WireGuard. PersistentKeepalive holds the connection open through your router.

Create `~/wireguard/wg-client` ([download](wg-client)):

```
#!/bin/sh
# Add a WireGuard client: wg-client NAME TUNNEL_IP SERVER_ADDRESS
set -eu
name=$1 addr=$2 endpoint=$3
cd /etc/wireguard
umask 077
wg genkey > "$name.key"
wg pubkey < "$name.key" > "$name.pub"

# the server learns the client's public key and its tunnel address
cat >> wg0.conf <<CONF

[Peer]
# $name
PublicKey = $(cat "$name.pub")
AllowedIPs = $addr/32
CONF

# the client gets its private key, and how to reach the server
cat > "$name.conf" <<CONF
[Interface]
PrivateKey = $(cat "$name.key")
Address = $addr/32

[Peer]
PublicKey = $(cat server.pub)
Endpoint = $endpoint:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
CONF
echo "created /etc/wireguard/$name.conf"
```

*On the server, as root (`sudo -i`):*

```bash
cat /home/YOUR_USER/wireguard/wg-client
install -m 700 /home/YOUR_USER/wireguard/wg-client /usr/local/sbin/
```

Add a laptop, and a phone. The last argument is the address the devices dial. In the lab, it's the server itself. Every device needs its own address in the tunnel: .2 for the laptop, .3 for the phone. Here's the laptop's config. Its private key, and the server's public key, address, and port.

```bash
wg-client laptop 10.8.0.2 SERVER_IP
wg-client phone 10.8.0.3 SERVER_IP
cat laptop.conf
```

Start the tunnel, and enable it at boot. wg show lists the interface and both devices, each with its public key and tunnel address. Nobody has connected yet, so there's no handshake.

```bash
systemctl enable --now wg-quick@wg0
wg show
```

> **Adding a device later:** run `wg-client` again, then `systemctl restart wg-quick@wg0`.
> **Firewall:** if you enabled ufw in episode 2, also run `sudo ufw allow 51820/udp`.

## Router and address

[▶ 1:56](https://youtu.be/tEgerx0zY3g?t=116s)

**From outside your home**

- Router: forward **UDP 51820** to the server
- Endpoint: your **public IP**, or a dynamic DNS name
- No public IP (CGNAT)? Use a small VPS as a relay

## Laptop

[▶ 2:21](https://youtu.be/tEgerx0zY3g?t=141s)

On the laptop, install wireguard-tools. That's for Linux. On Windows, Mac, Android and iPhone, there's an official WireGuard app, and it reads the same config file. Copy the laptop's config from the server, and name it homelab. That name becomes the interface name. It holds a private key, so only root can read it.

*On your workstation:*

```bash
sudo apt install -y wireguard-tools
ssh YOUR_USER@SERVER_IP sudo cat /etc/wireguard/laptop.conf | sudo tee /etc/wireguard/homelab.conf >/dev/null
sudo chmod 600 /etc/wireguard/homelab.conf
```

Away from home, your apps aren't at their local address. They're at the server's tunnel address. So point the names there. In the lab, my laptop sits right next to the server. Pointing the names at the tunnel address is how we pretend it's far away. And without the tunnel, the vault doesn't answer. curl gives up after five seconds.

```bash
sudo sed -i 's/^SERVER_IP /10.8.0.1 /' /etc/hosts
grep home.arpa /etc/hosts
curl -sSI -m 5 https://vault.home.arpa | head -1
```

Bring the tunnel up. wg-quick creates an interface called homelab, with a route for the tunnel network only. Now the vault answers, through WireGuard. wg show has a recent handshake, and the bytes going both ways.

```bash
sudo wg-quick up homelab
curl -sSI -m 5 https://vault.home.arpa | head -1
sudo wg show
```

> **Disconnect:** `sudo wg-quick down homelab`. **Connect at boot:** `sudo systemctl enable --now wg-quick@homelab`.
> On Windows and macOS, install the WireGuard app and import `laptop.conf` instead.

And in the browser, the vault and the Git server open over the tunnel, still with HTTPS from Caddy.

## Phone

[▶ 3:44](https://youtu.be/tEgerx0zY3g?t=224s)

For the phone, print its config as a QR code, right in the terminal. Install the WireGuard app, tap plus, and scan it. Give the tunnel a name, and switch it on. This code holds the phone's private key, so treat it like a password, and don't share a screenshot.

*On the server, as root (`sudo -i`):*

```bash
qrencode -t ansiutf8 < phone.conf
```

> **On the phone:** install *WireGuard* from Google Play or the App Store, tap **+**, choose
> **Scan from QR code** (Android: *Scan from QR code*; iPhone: *Create from QR code*), name the tunnel, and turn it on.
> The phone has no `/etc/hosts`, so the `home.arpa` names only work once a DNS server answers them for the tunnel
> (AdGuard Home, episode 8). Until then, the WireGuard app shows a handshake, and the server answers at `10.8.0.1`.
> Delete `phone.conf` and `phone.key` from the server once the phone is set up.

## Next steps

- One device, **one key**: run `wg-client` for each
- Lost a phone? Delete its `[Peer]` and restart
- Next episode: **AdGuard Home**, DNS for every device

## Links

- https://www.wireguard.com/quickstart/
- https://man7.org/linux/man-pages/man8/wg-quick.8.html
- https://www.wireguard.com/install/

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
