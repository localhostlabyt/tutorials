# Restic backup tutorial: back up Docker Compose apps to a NAS over SFTP (and restore)

▶ [Watch the video](https://youtu.be/j9rCJfzhKps) · EP 006 · Tested on: Debian 13 (trixie), restic 0.18.0, Gitea 28.2.0, Vaultwarden 1.37.5, Uptime Kuma 2.5.11

Restic backup tutorial for a homelab: back up every Docker Compose app on the server (Gitea with PostgreSQL, Vaultwarden, Uptime Kuma, Caddy) to another machine over SFTP, encrypted. A small script stops the apps for a consistent copy, a systemd timer runs it every night with a retention policy, and then we delete Gitea's data and restore it from the backup.

## Intro

[▶ 0:00](https://youtu.be/j9rCJfzhKps?t=0s)

**What we back up**

- Gitea: repositories and the **PostgreSQL** database
- Vaultwarden and Uptime Kuma data
- Caddy's data, with its **root certificate**
- All of it lives in `/home/admin`

## A place for the backups

[▶ 0:42](https://youtu.be/j9rCJfzhKps?t=42s)

Backups on the same disk die with the disk. So they go to another machine, over SSH. Any Linux box works: a NAS, a Raspberry Pi with a USB drive, or a cheap VPS. In the lab, my workstation plays the NAS. Create a user called backup, with an empty SSH directory.

*On your workstation:*

```bash
sudo useradd -m -s /bin/bash backup
sudo install -d -m 700 -o backup -g backup /home/backup/.ssh
```

## SSH key for the server

[▶ 1:02](https://youtu.be/j9rCJfzhKps?t=62s)

On the server, install restic from the Debian repository. Version 0.18. The backup runs as root, because the database files belong to other users. So root gets an SSH key, without a passphrase, for the nightly job. And a short name for the backup machine: nas.

*On the server:*

```bash
sudo apt install -y restic
restic version
sudo ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519 -q
echo "NAS_IP nas" | sudo tee -a /etc/hosts
```

Back on the workstation, copy root's public key from the server into the backup user's authorized_keys. Now the server can log in as backup, and only with that key.

*On your workstation:*

```bash
ssh YOUR_USER@SERVER_IP sudo cat /root/.ssh/id_ed25519.pub | sudo tee -a /home/backup/.ssh/authorized_keys
sudo chown backup: /home/backup/.ssh/authorized_keys
```

## Restic repository

[▶ 1:39](https://youtu.be/j9rCJfzhKps?t=99s)

On the server, switch to root. Trust the NAS host key once, and make a directory for the restic settings. Restic encrypts everything with a password. Generate a long random one. Save a copy of it somewhere else, like your password manager, and on paper. Without it, the backup is useless. Two settings: the repository, a folder called homelab on the NAS, over SFTP. And the password file. Load them, and create the repository.

Create `~/backup/env` ([download](env)):

```
export RESTIC_REPOSITORY=sftp:backup@nas:homelab
export RESTIC_PASSWORD_FILE=/etc/restic/password
```

*On the server, as root (`sudo -i`):*

```bash
ssh-keyscan -H nas >> ~/.ssh/known_hosts 2>/dev/null
install -d -m 700 /etc/restic
openssl rand -base64 32 > /etc/restic/password
install -m 600 /home/YOUR_USER/backup/env /etc/restic/env
cat /etc/restic/env
. /etc/restic/env
restic init
```

## Backup script

[▶ 2:19](https://youtu.be/j9rCJfzhKps?t=139s)

The backup script. It loads the settings, and lists the three compose projects. A database that's running can change in the middle of a copy. So the script stops the apps first. Uptime Kuma goes first, so the short pause doesn't wake you up. trap starts them again, even if the backup fails. Restic backs up the admin home directory. Then forget keeps seven daily, four weekly and six monthly snapshots, and prune frees the space. Install it as a root-only command.

Create `~/backup/homelab-backup` ([download](homelab-backup)):

```
#!/bin/sh
# Nightly homelab backup: stop the apps, copy everything with restic, start them again.
set -eu
. /etc/restic/env
APPS="/home/admin/uptime-kuma /home/admin/vaultwarden /home/admin/gitea"

start_apps() {
    for app in $APPS; do (cd "$app" && docker compose start); done
}
# stopped containers = consistent database files; start again even if the backup fails
trap start_apps EXIT
for app in $APPS; do (cd "$app" && docker compose stop); done

restic backup --tag nightly /home/admin
trap - EXIT
start_apps

# keep 7 daily, 4 weekly and 6 monthly snapshots, delete the rest
restic forget --tag nightly --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

*On the server, as root (`sudo -i`):*

```bash
cat /home/YOUR_USER/backup/homelab-backup
install -m 700 /home/YOUR_USER/backup/homelab-backup /usr/local/sbin/
```

Run it once by hand. The apps stop, restic copies the files, and the apps start again, all in a few seconds. Next time, restic only sends what changed, so nightly backups stay small and fast. The first snapshot is in the repository.

```bash
homelab-backup
. /etc/restic/env && restic snapshots
```

## Every night with systemd

[▶ 3:12](https://youtu.be/j9rCJfzhKps?t=192s)

A systemd timer runs it every night at three, plus a random delay of up to fifteen minutes. Persistent=true means a backup missed while the server was off runs at the next boot. The service next to it just runs the script. Copy both, and enable the timer. The next run is already scheduled.

Create `~/backup/homelab-backup.service` ([download](homelab-backup.service)):

```
[Unit]
Description=Homelab backup with restic
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/homelab-backup
```

Create `~/backup/homelab-backup.timer` ([download](homelab-backup.timer)):

```
[Unit]
Description=Nightly homelab backup

[Timer]
OnCalendar=*-*-* 03:00
RandomizedDelaySec=15min
Persistent=true

[Install]
WantedBy=timers.target
```

*On the server, as root (`sudo -i`):*

```bash
cp homelab-backup.service homelab-backup.timer /etc/systemd/system/
systemctl daemon-reload && systemctl enable --now homelab-backup.timer
systemctl list-timers homelab-backup.timer
```

> **Check the nightly runs:** `journalctl -u homelab-backup.service` shows the output of every run.
> To get an alert when a backup fails, add a *Push* monitor in Uptime Kuma (episode 5)
> and call its URL with `curl` at the end of the script.

## Disaster and restore

[▶ 3:45](https://youtu.be/j9rCJfzhKps?t=225s)

Now the part most people skip: the restore test. Let's lose Gitea completely. Take the project down, and delete its data and its database.

*On the server, as root (`sudo -i`):*

```bash
docker compose down
rm -rf gitea postgres
ls
```

The Git server is gone.

Restore the latest snapshot, only the Gitea folder, back to its original place. Restic puts back the files with their owners and permissions, so the database starts as if nothing happened. Start the project again.

```bash
. /etc/restic/env
restic restore latest --target / --include /home/YOUR_USER/gitea
ls
docker compose up -d
```

And the repository from episode one is back, with all its history.

## Next steps

- Keep the restic password **outside** the server
- Test a restore every few months
- Next episode: **WireGuard** remote access

## Links

- https://restic.readthedocs.io/en/stable/
- https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#sftp
- https://restic.readthedocs.io/en/stable/060_forget.html

---

`SERVER_IP` is your server's address and `YOUR_USER` your login on it (`NAS_IP`: the backup machine). These commands are extracted from the same script that recorded the video.
