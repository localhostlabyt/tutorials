# Your first 10 minutes on a new Linux server: SSH keys, firewall, auto-updates

▶ [Watch the video](https://youtu.be/ipP4cv9z8AY) · EP 002 · Tested on: Debian 13 (trixie), OpenSSH 10.0, ufw 0.36.2, unattended-upgrades 2.12

A fresh Debian 13 server, step by step: updates, SSH key login, password login disabled, a ufw firewall and automatic security updates.

## Intro

[▶ 0:00](https://youtu.be/ipP4cv9z8AY?t=0s)

**What you need**

- A fresh **Debian 13** server: a VPS or a VM
- Its IP address and a login with `sudo`
- A terminal on your own computer

## First login

[▶ 0:34](https://youtu.be/ipP4cv9z8AY?t=34s)

Log in with the password from your provider. The first time, SSH shows the server's fingerprint. Type yes to trust it. We're in. But password logins are exactly what bots on the internet try to guess, all day long. We'll switch them off in a few minutes.

*On your workstation:*

```bash
ssh YOUR_USER@SERVER_IP
```

## Updates

[▶ 0:49](https://youtu.be/ipP4cv9z8AY?t=49s)

First, refresh the package lists and upgrade everything that's installed. This server is already up to date. But images from providers are often weeks old, so usually there are security fixes waiting. If the kernel was updated, reboot when you're done.

*On the server:*

```bash
sudo apt update
sudo apt full-upgrade -y
```

## SSH keys

[▶ 1:05](https://youtu.be/ipP4cv9z8AY?t=65s)

Now, on your own computer, generate an SSH key pair. If you already have one, skip this step. For a real machine, protect the key with a passphrase. ssh-copy-id puts your public key on the server. This is the last time you type the server password. Log in again, and this time there's no password prompt. The key does the work.

*On your workstation:*

```bash
ssh-keygen -t ed25519 -C "you@workstation"
ssh-copy-id YOUR_USER@SERVER_IP
ssh YOUR_USER@SERVER_IP
```

## Lock down SSH

[▶ 1:32](https://youtu.be/ipP4cv9z8AY?t=92s)

With key login working, turn off passwords on the server. Put these settings in their own small file in sshd_config.d, instead of editing the main config file. That way, package updates never overwrite them. The name starts with 10 on purpose. SSH keeps the first value it reads, and these files are read in alphabetical order, before the main config. So ours wins. Check the syntax first, then reload SSH. Your current session stays open. sshd -T prints the settings that are actually in effect: passwords and root login are off.

*On the server:*

```bash
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\nPermitRootLogin no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E "^(passwordauth|kbdinteractiveauth|permitrootlogin)"
```

Test it from your own computer, before you close the server session. A login that skips the key is now refused. And with the key, everything still works.

*On your workstation:*

```bash
ssh -o PubkeyAuthentication=no YOUR_USER@SERVER_IP
ssh YOUR_USER@SERVER_IP hostname
```

## Firewall

[▶ 2:21](https://youtu.be/ipP4cv9z8AY?t=141s)

Next, a firewall. On Debian, the simplest option is ufw, a friendly front end to nftables. Block all incoming traffic by default, and allow all outgoing traffic. Allow SSH before you enable the firewall. In the other order, you'd cut off your own connection. Enable it, and confirm with y. Only SSH is open now. One warning for later: ports published by Docker bypass ufw rules, so check them separately when you add containers.

*On the server:*

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

## Automatic updates

[▶ 2:52](https://youtu.be/ipP4cv9z8AY?t=172s)

Finally, automatic security updates. Install unattended-upgrades. Some cloud images already include it, but a minimal install doesn't. This file turns it on: refresh the package lists and run the upgrade every day. A dry run shows where it takes updates from. By default, only from Debian and Debian-Security. And a systemd timer runs it once a day.

*On the server:*

```bash
sudo apt install -y unattended-upgrades
printf 'APT::Periodic::Update-Package-Lists "1";\nAPT::Periodic::Unattended-Upgrade "1";\n' | sudo tee /etc/apt/apt.conf.d/20auto-upgrades
sudo unattended-upgrade --dry-run --debug 2>&1 | grep "Allowed origins"
systemctl list-timers apt-daily-upgrade.timer --no-pager
```

## Next steps

- Reboot after kernel updates: `sudo reboot`
- Open ports only when needed: `sudo ufw allow 443/tcp`
- Next episode: **Caddy** with HTTPS in front of Gitea

## Links

- https://wiki.debian.org/UnattendedUpgrades
- https://manpages.debian.org/trixie/ufw/ufw.8.en.html
- https://manpages.debian.org/trixie/openssh-server/sshd_config.5.en.html

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it. These commands are extracted from the same script that recorded the video.
