# Updates

The server has two moving parts that age independently: the Debian base
system and the container images. Both are updated from cron, but they
cannot share a single job — the apt half needs root, and the Docker half
must **not** run as root.

## 1. Why Two Scripts and Two Crontabs

The compose file uses `~` in its volume paths, for example
`~/docker/config/plex:/config`. Docker Compose expands `~` from the
`$HOME` of whoever runs the command. Under `sudo` or in root's crontab
`$HOME` is `/root`, so the bind mount silently becomes
`/root/docker/config/plex` — an empty directory. The containers come
back up factory-reset.

You can see the difference yourself:

```bash
cd ~/docker && docker compose config | grep 'source:'   # /home/<user>/docker/...
sudo docker compose config | grep 'source:'             # /root/docker/...
```

!!! warning "This only affects commands that create containers"

    The backup script in [Backups](07-backups.md) runs as root, but it
    only calls `docker compose stop` and `start`. Those act on existing
    containers and never re-read the volume paths. `docker compose up -d`
    does re-read them, which is why the update job must run as your admin
    user — who is already in the `docker` group and therefore needs no
    `sudo` at all.

So:

- **apt** -> `/usr/local/sbin/apt-update-upgrade.sh`, root's crontab.
- **Docker** -> `/usr/local/bin/update-docker-media-server-stack.sh`,
  your user's crontab.

Both scripts live in root-owned directories so no unprivileged process
can modify a job that runs unattended.

## 2. The apt Script

```bash
sudo vim /usr/local/sbin/apt-update-upgrade.sh
```

```bash
#!/bin/bash
set -uo pipefail
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export DEBIAN_FRONTEND=noninteractive   # cron has no TTY, never prompt
export NEEDRESTART_MODE=a               # restart affected services without asking

echo "=== $(date '+%F %T') apt update ==="
apt-get update -qq || { echo "ERROR: update failed"; exit 1; }

echo "=== upgrade ==="
apt-get -y \
  -o Dpkg::Options::=--force-confdef \
  -o Dpkg::Options::=--force-confold \
  upgrade

echo "=== autoremove / autoclean ==="
apt-get -y --purge autoremove
apt-get -y autoclean

if [ -f /var/run/reboot-required ]; then
  echo "NOTE: reboot required -> $(tr '\n' ' ' < /var/run/reboot-required.pkgs 2>/dev/null)"
fi

echo "=== done ==="
```

Make it root-owned and executable:

```bash
sudo chown root:root /usr/local/sbin/apt-update-upgrade.sh
sudo chmod 755 /usr/local/sbin/apt-update-upgrade.sh
```

Why these details matter:

- `apt-get`, not `apt`. The `apt` command prints *"does not have a
  stable CLI interface"* when scripted, and its progress bars fill the
  log with control characters.

- `DEBIAN_FRONTEND=noninteractive` keeps dpkg from opening a dialog that
  nobody can answer. Without it a single config-file question blocks the
  job forever and holds the dpkg lock.

- `--force-confold` keeps your existing config files when a package
  ships a changed one. This is the conservative choice: your edits
  survive, and the new version is left as a `.dpkg-dist` file next to it.

- `upgrade`, not `full-upgrade`. Plain `upgrade` never removes a package
  to satisfy a dependency. Run `sudo apt-get full-upgrade` by hand when a
  Debian point release needs it, so you can read what it wants to do.

- `NEEDRESTART_MODE=a` tells Debian's `needrestart` to restart affected
  services automatically instead of asking. Leave it out and library
  updates apply only after the next reboot.

## 3. The Docker Script

```bash
sudo vim /usr/local/bin/update-docker-media-server-stack.sh
```

```bash
#!/bin/bash
set -uo pipefail
export PATH=/usr/local/bin:/usr/bin:/bin

COMPOSE_DIR="/home/<user>/docker"

cd "$COMPOSE_DIR" || { echo "ERROR: cannot cd to $COMPOSE_DIR"; exit 1; }

echo "=== $(date '+%F %T') docker compose pull ==="
if ! docker compose pull; then
  echo "ERROR: pull failed, leaving containers untouched."; exit 1
fi

echo "=== recreating changed containers ==="
docker compose up -d

echo "=== pruning old images ==="
docker image prune -f

echo "=== done ==="
```

```bash
sudo chown root:root /usr/local/bin/update-docker-media-server-stack.sh
sudo chmod 755 /usr/local/bin/update-docker-media-server-stack.sh
```

What the script does:

- Aborts if `pull` fails. A partial pull would otherwise leave `up -d`
  recreating some services on new images and others on old ones, which is
  the hardest state to debug.

- Only recreates containers whose image actually changed. Services with
  an unchanged digest are left running untouched.

- Prunes dangling images, so the replaced layers do not accumulate on the
  system disk.

## 4. Manual Test

Run both once by hand before scheduling them. Note that the Docker
script is called **without** `sudo`:

```bash
sudo /usr/local/sbin/apt-update-upgrade.sh
/usr/local/bin/update-docker-media-server-stack.sh
```

Then confirm the stack is healthy and the volume paths are still correct:

```bash
docker compose -f ~/docker/docker-compose.yml ps
docker compose -f ~/docker/docker-compose.yml config | grep 'source:'
```

## 5. Schedule with Cron

The backup job from [Backups](07-backups.md) runs Mondays at 03:00 and
stops every container while it works. The two update jobs are arranged
around it so that no two jobs interfere:

| Time | Job | Runs as |
| --- | --- | --- |
| Mondays 03:00 | `backup-docker-config.sh` | root |
| Tue–Sun 04:30 | `apt-update-upgrade.sh` | root |
| Mondays 06:00 | `update-docker-media-server-stack.sh` | admin user |

Root's crontab, for apt:

```bash
sudo crontab -e
```

```conf
30 4 * * 0,2-6 /usr/local/sbin/apt-update-upgrade.sh >> /var/log/apt-update-upgrade.log 2>&1
```

The day field `0,2-6` means every day except Monday. Cron counts `0` as
Sunday and `1` as Monday, so `0,2-6` is Sunday plus Tuesday through
Saturday. Monday is left free for the backup and the container update.

Your own crontab, for Docker — **no `sudo`**, otherwise you edit root's
crontab and reintroduce the `$HOME` problem from section 1:

```bash
mkdir -p ~/logs
crontab -e
```

```conf
0 6 * * 1 /usr/local/bin/update-docker-media-server-stack.sh >> /home/<user>/logs/update-docker-media-server-stack.log 2>&1
```

## 6. Notes

- **Reboots stay manual.** The apt script only writes a `reboot required`
  line into its log; it never reboots on its own. Check with
  `ls /var/run/reboot-required`. If you do want it automatic, a separate
  root cron entry such as
  `0 6 * * 1 [ -f /var/run/reboot-required ] && /sbin/reboot` keeps the
  decision visible in one place.

- **Mail on failure.** Adding `MAILTO=you@example.com` at the top of a
  crontab makes cron mail you any output a job produces. This only helps
  if the server can actually send mail; otherwise the log files are your
  only channel.

- **`unattended-upgrades` is the alternative.** Debian ships a dedicated
  package for the apt half that handles locking, retries and mail
  reporting for you. The cron script above is used here because it keeps
  the package and container updates in one place, with one log style.

- **Pinning image tags.** If an update ever breaks a service, replace
  `:latest` with a concrete version tag in the compose file for that one
  service. The update job will then keep it where it is until you change
  the tag back.
