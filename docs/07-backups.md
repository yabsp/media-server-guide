# Backups

The media server configuration lives in a single folder (`~/docker`),
which contains the compose file and the `config` directory for every
service. Backing up this folder is enough to restore the whole stack.

The main risk is database corruption. Services like Sonarr and Radarr
use SQLite, and copying a live database can produce a broken backup. To
avoid this we stop the containers, take a consistent snapshot, start the
containers again, and only then compress the snapshot to the NAS.

## 1. Persistent NAS Mount

To make sure the backup target is always available, we mount a NAS share
over CIFS at boot.

### Credentials File

Store the NAS login in a root-owned file so the password is not written
into `/etc/fstab`.

```bash
sudo vim /etc/.diskstation_creds
```

```conf
username=<nas-user>
password=<nas-password>
```

Lock down the permissions so only root can read it:

```bash
sudo chown root:root /etc/.diskstation_creds
sudo chmod 600 /etc/.diskstation_creds
```

### fstab Entry

Add the mount to `/etc/fstab`:

```conf
//192.168.x.x/<share> /mnt/docker_backups cifs credentials=/etc/.nas_creds,uid=1000,gid=1000,dir_mode=0755,file_mode=0644,nofail,_netdev,vers=3.0 0 0
```

Key options:

- `nofail`: the boot continues even if the NAS is offline.

- `_netdev`: waits for the network before mounting.

- `vers=3.0`: pins the SMB protocol version to avoid negotiation errors.

Test the mount without rebooting:

```bash
sudo mkdir -p /mnt/docker_backups
sudo mount -a
mountpoint /mnt/docker_backups
```

## 2. Why the Backup Runs as Root

The service containers run as UID 13000 (see chapter
[User & Network](02-network.md)). Some applications, Plex in particular,
store secrets such as `Preferences.xml` and API tokens with mode `0600`,
readable only by their owner. A normal admin account cannot read these
files even when it is a member of the same group, because the group read
bit is not set.

For that reason the backup script must run as root. Root ignores
permission bits and can read every container's config regardless of which
UID owns it. This is the standard approach for backing up Docker volumes.

## 3. The Backup Script

Create the script in a root-owned location so no unprivileged user can
edit a job that runs as root.

```bash
sudo vim /usr/local/sbin/backup-docker-config.sh
```

```bash
#!/bin/bash
set -uo pipefail
export PATH=/usr/local/bin:/usr/bin:/bin

# --- CONFIG: adjust to your setup ---
COMPOSE_DIR="/home/<user>/docker"          # holds docker-compose.yml + ./config
BACKUP_DEST="/mnt/docker_backups"          # the CIFS mount
STAGING="/home/<user>/.backup-staging"     # local temp, needs room for one config copy
# ------------------------------------

DATE=$(date +%Y-%m-%d_%H%M)
ARCHIVE_NAME="mediaserver_cfg_$DATE.tar.gz"
COMPOSE_FILE=$(ls "$COMPOSE_DIR"/docker-compose.y*ml 2>/dev/null | head -1)

# 1. Refuse to run unless the NAS is mounted, otherwise tar would fill the root disk
if ! mountpoint -q "$BACKUP_DEST"; then
  echo "ERROR: $BACKUP_DEST is not mounted. Aborting."; exit 1
fi

cd "$COMPOSE_DIR" || { echo "ERROR: cannot cd to $COMPOSE_DIR"; exit 1; }

# 2. Always bring containers back up, even if the script fails or is killed
restart() { echo "Starting containers..."; docker compose start; }
trap restart EXIT

# 3. Stop containers, snapshot config locally, restart as soon as possible
echo "Stopping containers..."
docker compose stop

echo "Snapshotting config to local staging..."
mkdir -p "$STAGING"
rsync -a --delete "$COMPOSE_DIR/config/" "$STAGING/config/"
cp "$COMPOSE_FILE" "$STAGING/"

echo "Restarting containers (downtime ends here)..."
docker compose start
trap - EXIT   # already restarted, drop the safety net

# 4. Compress the local snapshot to the NAS while containers run
echo "Archiving to $BACKUP_DEST/$ARCHIVE_NAME ..."
if tar -czf "$BACKUP_DEST/$ARCHIVE_NAME" -C "$STAGING" config "$(basename "$COMPOSE_FILE")"; then
  echo "Archive OK."
  # 5. Prune old backups only after a successful new one
  find "$BACKUP_DEST" -name "mediaserver_cfg_*.tar.gz" -type f -mtime +30 -delete
else
  echo "ERROR: tar failed, keeping old backups."
fi

echo "Done: $ARCHIVE_NAME"
```

What the script does:

- Copies to local staging first, so the containers are only stopped for
  the duration of an incremental rsync (seconds after the first run).

- Uses a `trap` so the containers always restart, even on error or if the
  script is interrupted during the stopped window.

- Prunes backups older than 30 days, but only after a new archive was
  written successfully.

Make the script root-owned and executable:

```bash
sudo chown root:root /usr/local/sbin/backup-docker-config.sh
sudo chmod 755 /usr/local/sbin/backup-docker-config.sh
```
**Note** that alternatively the script can be located in the user's home directory, but better is a root owned directory.

## 4. Manual Test

Run the script once by hand before scheduling it:

```bash
sudo /usr/local/sbin/backup-docker-config.sh
```

Confirm the archive was written and is readable:

```bash
sudo ls -lh /mnt/docker_backups/
sudo tar -tzf /mnt/docker_backups/mediaserver_cfg_*.tar.gz | head
```

While it runs, you can watch the containers stop and restart in a second
terminal:

```bash
watch -n1 'docker compose -f /home/<user>/docker/docker-compose.yml ps'
```

## 5. Schedule with Cron

Because the job runs as root, install it in root's crontab:

```bash
sudo crontab -e
```

Run it daily at 03:00:

```conf
0 3 * * * /usr/local/sbin/backup-docker-config.sh >> /var/log/backup-docker-config.log 2>&1
```

Other useful times:

- `0 3 * * 1` -> Mondays 03:00
- `0 3 1,15 * *` -> twice a month (1st and 15th)
- `0 3 1 * *` -> monthly, 1st of the month

Notes:

- The script sets its own `PATH`, since cron does not load the normal
  environment.

- `>>` appends to the log so you keep a history instead of overwriting it
  each night.

- After the first scheduled run, check the log with
  `sudo tail -n 20 /var/log/backup-docker-config.log`.

## 6. Security Notes

- A root cron job is only as safe as the script it runs. Keep the script
  in a root-owned directory (`/usr/local/sbin`) so no unprivileged user
  can modify a job that runs as root.

- The first backup copies the full config and can be large. Every run
  after that is incremental, so the stopped window stays short.
