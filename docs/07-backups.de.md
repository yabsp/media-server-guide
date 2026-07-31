# Backups

Die Media-Server-Konfiguration liegt in einem einzigen Ordner (`~/docker`), der
die Compose-Datei und das `config`-Verzeichnis jedes Dienstes enthält. Das
Sichern dieses Ordners genügt, um den gesamten Stack wiederherzustellen.

Das Hauptrisiko ist Datenbankkorruption. Dienste wie Sonarr und Radarr verwenden
SQLite, und das Kopieren einer laufenden Datenbank kann ein defektes Backup
erzeugen. Um das zu vermeiden, stoppen wir die Container, erstellen einen
konsistenten Snapshot, starten die Container wieder und komprimieren den Snapshot
erst dann auf das NAS.

## 1. Dauerhaftes NAS-Einhängen

Um sicherzustellen, dass das Backup-Ziel immer verfügbar ist, hängen wir eine
NAS-Freigabe beim Booten über CIFS ein.

### Zugangsdaten-Datei

Speichere die NAS-Anmeldedaten in einer root-eigenen Datei, damit das Passwort
nicht in `/etc/fstab` geschrieben wird.

```bash
sudo vim /etc/.diskstation_creds
```

```conf
username=<nas-user>
password=<nas-password>
```

Sperre die Berechtigungen, sodass nur root sie lesen kann:

```bash
sudo chown root:root /etc/.diskstation_creds
sudo chmod 600 /etc/.diskstation_creds
```

### fstab-Eintrag

Füge das Einhängen zu `/etc/fstab` hinzu:

```conf
//192.168.x.x/<share> /mnt/docker_backups cifs credentials=/etc/.diskstation_creds,uid=1000,gid=1000,dir_mode=0755,file_mode=0644,nofail,_netdev,vers=3.0 0 0
```

Wichtige Optionen:

- `nofail`: Der Bootvorgang wird fortgesetzt, auch wenn das NAS offline ist.

- `_netdev`: Wartet auf das Netzwerk, bevor eingehängt wird.

- `vers=3.0`: Legt die SMB-Protokollversion fest, um Aushandlungsfehler zu vermeiden.

Teste das Einhängen ohne Neustart:

```bash
sudo mkdir -p /mnt/docker_backups
sudo mount -a
mountpoint /mnt/docker_backups
```

## 2. Warum das Backup als Root läuft

Die Dienst-Container laufen als UID 13000 (siehe Kapitel
[Benutzer & Netzwerk](02-network.md)). Einige Anwendungen, insbesondere Plex,
speichern Geheimnisse wie `Preferences.xml` und API-Tokens mit dem Modus `0600`,
lesbar nur für ihren Eigentümer. Ein normales Admin-Konto kann diese Dateien
nicht lesen, selbst wenn es Mitglied derselben Gruppe ist, weil das
Gruppen-Lesebit nicht gesetzt ist.

Aus diesem Grund muss das Backup-Skript als root laufen. Root ignoriert
Berechtigungsbits und kann die Konfiguration jedes Containers lesen, unabhängig
davon, welcher UID sie gehört. Dies ist das übliche Vorgehen zum Sichern von
Docker-Volumes.

## 3. Das Backup-Skript

Erstelle das Skript an einem root-eigenen Ort, damit kein unprivilegierter
Benutzer einen Job bearbeiten kann, der als root läuft.

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

Was das Skript tut:

- Kopiert zuerst in ein lokales Staging, sodass die Container nur für die Dauer
  eines inkrementellen rsync gestoppt werden (nach dem ersten Durchlauf nur
  Sekunden).

- Verwendet ein `trap`, damit die Container immer neu starten, auch bei einem
  Fehler oder wenn das Skript während des Stopp-Fensters unterbrochen wird.

- Löscht Backups, die älter als 30 Tage sind, aber erst, nachdem ein neues
  Archiv erfolgreich geschrieben wurde.

Mache das Skript root-eigen und ausführbar:

```bash
sudo chown root:root /usr/local/sbin/backup-docker-config.sh
sudo chmod 755 /usr/local/sbin/backup-docker-config.sh
```
**Hinweis:** Das Skript kann alternativ im Home-Verzeichnis des Benutzers liegen, ein root-eigenes Verzeichnis ist jedoch vorzuziehen.

## 4. Manueller Test

Führe das Skript vor dem Einplanen einmal von Hand aus:

```bash
sudo /usr/local/sbin/backup-docker-config.sh
```

Bestätige, dass das Archiv geschrieben wurde und lesbar ist:

```bash
sudo ls -lh /mnt/docker_backups/
sudo tar -tzf /mnt/docker_backups/mediaserver_cfg_*.tar.gz | head
```

Während es läuft, kannst du in einem zweiten Terminal beobachten, wie die
Container stoppen und neu starten:

```bash
watch -n1 'docker compose -f /home/<user>/docker/docker-compose.yml ps'
```

## 5. Mit Cron einplanen

Da der Job als root läuft, installiere ihn in der Crontab von root:

```bash
sudo crontab -e
```

Führe ihn täglich um 03:00 Uhr aus:

```conf
0 3 * * * /usr/local/sbin/backup-docker-config.sh >> /var/log/backup-docker-config.log 2>&1
```

Weitere nützliche Zeiten:

- `0 3 * * 1` -> Montags 03:00 Uhr
- `0 3 1,15 * *` -> zweimal im Monat (am 1. und 15.)
- `0 3 1 * *` -> monatlich, am 1. des Monats

Hinweise:

- Das Skript setzt seinen eigenen `PATH`, da Cron die normale Umgebung nicht
  lädt.

- `>>` hängt an das Log an, sodass du eine Historie behältst, statt es jede
  Nacht zu überschreiben.

- Prüfe nach dem ersten geplanten Durchlauf das Log mit
  `sudo tail -n 20 /var/log/backup-docker-config.log`.

## 6. Sicherheitshinweise

- Ein Root-Cron-Job ist nur so sicher wie das Skript, das er ausführt. Halte das
  Skript in einem root-eigenen Verzeichnis (`/usr/local/sbin`), damit kein
  unprivilegierter Benutzer einen als root laufenden Job verändern kann.

- Das erste Backup kopiert die vollständige Konfiguration und kann gross sein.
  Jeder Durchlauf danach ist inkrementell, sodass das Stopp-Fenster kurz bleibt.
