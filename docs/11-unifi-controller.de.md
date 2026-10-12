# UniFi Controller (Docker)

Das WLAN im Haus läuft über zwei Ubiquiti **UAP-AC-PRO** (`UG-AP` und
`Licht-AP`), beide per Netzwerkkabel angeschlossen. UniFi-Geräte
haben keine eigene Weboberfläche; sie werden über einen zentralen
**Controller** (UniFi Network Application) eingerichtet. Diesen
betreiben wir als weiteren Container auf dem Mediaserver.

Der Controller wird nur für Einrichtung, Änderungen und Firmware-Updates
gebraucht. Das WLAN läuft weiter, wenn der Server aus ist.

Diese Seite beschreibt nur die Installation des Controllers auf dem
Debian-Server. Alles, was die Geräte selbst betrifft (einrichten,
zurücksetzen, betreiben), steht unter
[UniFi Access Points](12-unifi-access-points.md).

!!! warning "Standalone-Controller läuft aus"

    Die Oberfläche (Version 10.0) meldet beim Öffnen, dass künftige
    Network-Versionen nur noch für **UniFi OS Server** erscheinen. Der
    hier beschriebene Container funktioniert weiterhin, erhält aber
    irgendwann keine neuen Versionen mehr. Ein Umzug ist per
    Backup/Restore möglich.

## 1. Controller in Docker Compose

### Ports

| Port | Zweck |
| --- | --- |
| `8443/tcp` | Weboberfläche |
| `8081/tcp` | Inform (Access Point -> Controller) |
| `3478/udp` | STUN |
| `10001/udp` | Geräteerkennung |

Der Standard-Inform-Port wäre `8080`. Dieser ist auf dem Server bereits
durch einen anderen Container belegt, deshalb verwenden wir `8081`.

### Init-Skript für die Datenbank

Der Controller braucht eine MongoDB mit einem eigenen Benutzer. Das
Skript legt ihn beim allerersten Start der Datenbank an.

```bash
mkdir -p ~/docker/config/unifi ~/docker/config/unifi-db
vim ~/docker/config/unifi-db/init-mongo.sh
```

```bash
#!/bin/bash
mongosh <<EOF
use ${MONGO_AUTHSOURCE}
db.auth("${MONGO_INITDB_ROOT_USERNAME}", "${MONGO_INITDB_ROOT_PASSWORD}")
db.createUser({
  user: "${MONGO_USER}",
  pwd: "${MONGO_PASS}",
  roles: [
    { db: "${MONGO_DBNAME}", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_stat", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_audit", role: "dbOwner" }
  ]
})
EOF
```

### Dienste

Ergänze in `~/docker/docker-compose.yml` unter `services:`:

```yaml
  # ==========================================
  # UNIFI - Datenbank
  # ==========================================
  unifi-db:
    image: mongo:7.0
    container_name: unifi-db
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=<root-password>
      - MONGO_USER=unifi
      - MONGO_PASS=<unifi-password>
      - MONGO_DBNAME=unifi
      - MONGO_AUTHSOURCE=admin
    volumes:
      - ~/docker/config/unifi-db/data:/data/db
      - ~/docker/config/unifi-db/init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro
    restart: unless-stopped

  # ==========================================
  # UNIFI - Controller
  # ==========================================
  unifi:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi
    depends_on:
      - unifi-db
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
      - MONGO_USER=unifi
      - MONGO_PASS=<unifi-password>
      - MONGO_HOST=unifi-db
      - MONGO_PORT=27017
      - MONGO_DBNAME=unifi
      - MONGO_AUTHSOURCE=admin
    volumes:
      - ~/docker/config/unifi:/config
    ports:
      - 8443:8443       # Weboberfläche
      - 8081:8081       # Inform
      - 3478:3478/udp   # STUN
      - 10001:10001/udp # Geräteerkennung
    restart: unless-stopped
```

!!! warning "Drei Stolpersteine"

    - **Passwörter nur aus Buchstaben und Ziffern.** Sonderzeichen wie
      `:`, `@` oder `/` machen die Verbindungs-URL zur Datenbank
      ungültig. Erzeuge sie mit `openssl rand -hex 16`.
    - **Kein `network_mode: host`.** Im Host-Netzwerk kann der Controller
      den Namen `unifi-db` nicht auflösen.
    - **`MONGO_PASS` an beiden Stellen identisch.** Das Passwort wird nur
      beim ersten Start übernommen. Um es später zu ändern, müssen
      `~/docker/config/unifi-db/data` und `~/docker/config/unifi`
      gelöscht werden.

### Erster Start

Zuerst die Datenbank, dann der Controller. Der Inform-Port wird in der
Konfigurationsdatei gesetzt, die erst beim ersten Start entsteht.

```bash
cd ~/docker
docker compose up -d unifi-db
docker compose up -d unifi

# Rund eine Minute warten, dann den Inform-Port setzen
docker compose stop unifi
echo "unifi.http.port=8081" | sudo tee -a ~/docker/config/unifi/data/system.properties
docker compose up -d unifi
```

Prüfe von einem anderen Rechner aus, ob der Inform-Port antwortet. Ein
`HTTP/1.1 400` ist die erwartete Antwort.

```bash
curl -i http://192.168.0.50:8081/inform
```

Die Weboberfläche liegt unter `https://192.168.0.50:8443`. Die
Zertifikatswarnung im Browser ist normal. Lege im Assistenten ein
lokales Konto an.

Weiter geht es mit den Grundeinstellungen und dem ersten Gerät unter
[UniFi Access Points](12-unifi-access-points.md#8-grundeinstellungen-im-controller).

## 2. Fehlerbehebung

| Symptom | Ursache und Abhilfe |
| --- | --- |
| `Defined MONGO_HOST unifi-db is not reachable` | `unifi` läuft mit `network_mode: host` oder in einem anderen Compose-Projekt als `unifi-db`. |
| `The connection string contains invalid user information` | Sonderzeichen in `MONGO_PASS`. Neues Passwort setzen und beide Datenverzeichnisse löschen. |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Port bereits belegt. Inform-Port wie oben auf `8081` legen. |
