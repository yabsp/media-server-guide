# Aktualisierungen

Der Server hat zwei Teile, die unabhängig voneinander altern: das
Debian-Basissystem und die Container-Images. Beide werden über Cron
aktualisiert, können sich aber keinen gemeinsamen Job teilen — der
apt-Teil benötigt root, und der Docker-Teil darf **nicht** als root
laufen.

## 1. Warum zwei Skripte und zwei Crontabs

Die Compose-Datei verwendet `~` in ihren Volume-Pfaden, zum Beispiel
`~/docker/config/plex:/config`. Docker Compose löst `~` über das `$HOME`
desjenigen auf, der den Befehl ausführt. Unter `sudo` oder in der Crontab
von root ist `$HOME` gleich `/root`, sodass das Bind-Mount stillschweigend
zu `/root/docker/config/plex` wird — einem leeren Verzeichnis. Die
Container starten dann mit Werkseinstellungen neu.

Den Unterschied kannst du selbst sehen:

```bash
cd ~/docker && docker compose config | grep 'source:'   # /home/<user>/docker/...
sudo docker compose config | grep 'source:'             # /root/docker/...
```

!!! warning "Betrifft nur Befehle, die Container erzeugen"

    Das Backup-Skript aus [Datensicherung](07-backups.md) läuft als root,
    ruft aber nur `docker compose stop` und `start` auf. Diese wirken auf
    bestehende Container und lesen die Volume-Pfade nie neu ein.
    `docker compose up -d` liest sie hingegen neu ein. Deshalb muss der
    Update-Job als dein Admin-Benutzer laufen — der ohnehin in der Gruppe
    `docker` ist und daher gar kein `sudo` braucht.

Also:

- **apt** -> `/usr/local/sbin/apt-update-upgrade.sh`, Crontab von root.
- **Docker** -> `/usr/local/bin/update-docker-media-server-stack.sh`,
  Crontab deines Benutzers.

Beide Skripte liegen in root-eigenen Verzeichnissen, damit kein
unprivilegierter Prozess einen unbeaufsichtigt laufenden Job verändern
kann.

## 2. Das apt-Skript

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

Mache es root-eigen und ausführbar:

```bash
sudo chown root:root /usr/local/sbin/apt-update-upgrade.sh
sudo chmod 755 /usr/local/sbin/apt-update-upgrade.sh
```

Warum diese Details wichtig sind:

- `apt-get`, nicht `apt`. Der Befehl `apt` gibt beim Einsatz in Skripten
  *"does not have a stable CLI interface"* aus, und seine Fortschritts-
  balken füllen das Log mit Steuerzeichen.

- `DEBIAN_FRONTEND=noninteractive` verhindert, dass dpkg einen Dialog
  öffnet, den niemand beantworten kann. Ohne diese Variable blockiert
  eine einzige Rückfrage zu einer Konfigurationsdatei den Job dauerhaft
  und hält dabei die dpkg-Sperre.

- `--force-confold` behält deine bestehenden Konfigurationsdateien, wenn
  ein Paket eine geänderte Version mitbringt. Das ist die konservative
  Wahl: Deine Anpassungen bleiben erhalten, und die neue Version wird als
  `.dpkg-dist`-Datei daneben abgelegt.

- `upgrade`, nicht `full-upgrade`. Das einfache `upgrade` entfernt nie
  ein Paket, um eine Abhängigkeit aufzulösen. Führe
  `sudo apt-get full-upgrade` von Hand aus, wenn ein Debian-Point-Release
  das erfordert, damit du nachlesen kannst, was es vorhat.

- `NEEDRESTART_MODE=a` weist Debians `needrestart` an, betroffene Dienste
  automatisch neu zu starten statt nachzufragen. Ohne diese Variable
  greifen Bibliotheks-Updates erst nach dem nächsten Neustart.

## 3. Das Docker-Skript

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

Was das Skript tut:

- Bricht ab, wenn `pull` fehlschlägt. Ein unvollständiger Pull würde
  sonst dazu führen, dass `up -d` einige Dienste auf neuen und andere auf
  alten Images neu erstellt — der Zustand, der am schwersten zu
  diagnostizieren ist.

- Erstellt nur Container neu, deren Image sich tatsächlich geändert hat.
  Dienste mit unverändertem Digest laufen unberührt weiter.

- Löscht verwaiste Images, damit sich die ersetzten Layer nicht auf der
  Systemplatte ansammeln.

## 4. Manueller Test

Führe beide vor dem Einplanen einmal von Hand aus. Beachte, dass das
Docker-Skript **ohne** `sudo` aufgerufen wird:

```bash
sudo /usr/local/sbin/apt-update-upgrade.sh
/usr/local/bin/update-docker-media-server-stack.sh
```

Bestätige danach, dass der Stack gesund ist und die Volume-Pfade noch
stimmen:

```bash
docker compose -f ~/docker/docker-compose.yml ps
docker compose -f ~/docker/docker-compose.yml config | grep 'source:'
```

## 5. Mit Cron einplanen

Der Backup-Job aus [Datensicherung](07-backups.md) läuft montags um 03:00
Uhr und stoppt währenddessen alle Container. Die beiden Update-Jobs
werden so darum herum gelegt, dass sich nie zwei Jobs sich gegenseitig beeinträchtigen:

| Zeit | Job | Läuft als |
| --- | --- | --- |
| montags 03:00 | `backup-docker-config.sh` | root |
| Di–So 04:30 | `apt-update-upgrade.sh` | root |
| montags 06:00 | `update-docker-media-server-stack.sh` | Admin-Benutzer |

Die Crontab von root, für apt:

```bash
sudo crontab -e
```

```conf
30 4 * * 0,2-6 /usr/local/sbin/apt-update-upgrade.sh >> /var/log/apt-update-upgrade.log 2>&1
```

Das Tagesfeld `0,2-6` bedeutet jeden Tag ausser Montag. Cron zählt `0`
als Sonntag und `1` als Montag, `0,2-6` ist also Sonntag plus Dienstag
bis Samstag. Der Montag bleibt für das Backup und das Container-Update
frei.

Deine eigene Crontab, für Docker — **ohne `sudo`**, sonst bearbeitest du
die Crontab von root und handelst dir das `$HOME`-Problem aus Abschnitt 1
wieder ein:

```bash
mkdir -p ~/logs
crontab -e
```

```conf
0 6 * * 1 /usr/local/bin/update-docker-media-server-stack.sh >> /home/<user>/logs/update-docker-media-server-stack.log 2>&1
```

Warum das Container-Update am Montag um 06:00 Uhr liegt, drei Stunden
nach dem Backup:

- **Es hat immer ein frisches Backup im Rücken.** Das Backup dauert rund
  elf Minuten und ist um 06:00 Uhr längst fertig. Macht ein neues Image
  einen Dienst kaputt, stammt der Wiederherstellungspunkt von diesem
  Morgen und nicht von letzter Woche.

- **Um 06:00 Uhr an einem Montag streamt niemand.** `docker compose up -d`
  startet jeden Container neu, dessen Image sich geändert hat, und
  unterbricht damit jede laufende Wiedergabe.

Warum Docker-Updates wöchentlich und nicht täglich: Jedes Image in der
Compose-Datei ist auf `:latest` festgelegt, jeder Pull ist also ein
ungeprüfter Versionssprung über den ganzen Stack. Wöchentlich bedeutet
ein Wiederherstellungsfenster pro Woche statt sieben, gebunden an den
Tag, an dem ein frisches Backup existiert.

Prüfe nach den ersten geplanten Durchläufen die Logs:

```bash
sudo tail -n 30 /var/log/apt-update-upgrade.log
tail -n 30 ~/logs/update-docker-media-server-stack.log
```

## 6. Hinweise

- **Neustarts bleiben manuell.** Das apt-Skript schreibt nur eine Zeile
  `reboot required` in sein Log; es startet den Server nie von selbst
  neu. Prüfe das mit `ls /var/run/reboot-required`. Wenn du es doch
  automatisch willst, hält ein separater Root-Cron-Eintrag wie
  `0 6 * * 1 [ -f /var/run/reboot-required ] && /sbin/reboot` die
  Entscheidung an einer sichtbaren Stelle.

- **Benachrichtigung bei Fehlern.** Ein `MAILTO=du@example.com` am Anfang
  einer Crontab lässt Cron dir jede Ausgabe eines Jobs per Mail schicken.
  Das hilft nur, wenn der Server tatsächlich Mail versenden kann;
  andernfalls sind die Log-Dateien dein einziger Kanal.

- **`unattended-upgrades` ist die Alternative.** Debian liefert für den
  apt-Teil ein eigenes Paket mit, das Sperren, Wiederholungsversuche und
  Mail-Berichte für dich übernimmt. Das Cron-Skript oben wird hier
  verwendet, weil es Paket- und Container-Updates an einem Ort und mit
  einem einheitlichen Log-Stil hält.

- **Image-Tags festlegen.** Falls ein Update einmal einen Dienst kaputt
  macht, ersetze `:latest` in der Compose-Datei für genau diesen Dienst
  durch einen konkreten Versions-Tag. Der Update-Job lässt ihn dann dort,
  bis du den Tag wieder änderst.
