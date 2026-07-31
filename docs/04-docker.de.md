# Docker & Dienste

Wir verwenden **Docker**, um Anwendungen zu containerisieren. Das hält das
Basis-Betriebssystem sauber und ermöglicht einfache Updates und Backups. Wir
orchestrieren diese Container mit **Docker Compose**, womit wir die gesamte
Serverkonfiguration in einer einzigen Textdatei definieren können.

## Installation

Wir installieren die offizielle Docker Engine aus dem Docker-Repository, um
sicherzustellen, dass wir die neueste Version haben (die Standard-Repositories
von Debian sind oft veraltet). Der gesamte Vorgang ist in der [Docker-Dokumentation](https://docs.docker.com/engine/install/debian/) sehr gut beschrieben.

### Benutzerberechtigungen

Standardmässig erfordern Docker-Befehle `sudo`. Um das zu beheben, fügen wir
unseren Admin-Benutzer der `docker`-Gruppe hinzu.

## Verzeichnisstruktur

Wir behalten alle Docker-Konfigurationen in unserem Home-Verzeichnis. Das macht
das Sichern der Serverkonfiguration einfach: Es genügt, diesen einen Ordner zu
sichern.

```bash
# Das Haupt-Docker-Verzeichnis erstellen 
mkdir -p ~/docker

# Ein Verzeichnis für die Compose-Datei erstellen 
cd ~/docker 
touch docker-compose.yml
```

## Die Docker-Compose-Datei

Dies ist das Herzstück des Servers. Wir bearbeiten die Datei:

```bash
vim ~/docker/docker-compose.yml
```

Nachfolgend die Konfigurationsvorlage, optimiert für:

- **Intel i5-12600K:** QuickSync-Transcoding aktiviert.

- **Dienstbenutzer:** Verwendung von UID 13000, wie im Kapitel [Benutzer & Netzwerk](02-network.md) definiert

- **Hardlinks:** Optimierte Volume-Zuordnung für sofortige Verschiebungen.

- **Sicherheit:** Plex erhält Nur-Lese-Zugriff.

```yaml
services:
  # ==========================================
  # PLEX - Media Server (Intel QuickSync)
  # ==========================================
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
      - VERSION=docker
    volumes:
      - ~/docker/config/plex:/config
      - /mnt/data/media:/data/media:ro
      # TRANSCODING: RAM (/dev/shm)
      - /dev/shm:/transcode
    devices:
      - /dev/dri:/dev/dri
    restart: unless-stopped

  # ==========================================
  # SONARR HD (1080p)
  # ==========================================
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/sonarr:/config
      - /mnt/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped

  # ==========================================
  # SONARR 4K
  # ==========================================
  sonarr4k:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr4k
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/sonarr4k:/config
      - /mnt/data:/data
    ports:
      - 8990:8989
    restart: unless-stopped

  # ==========================================
  # RADARR HD (1080p)
  # ==========================================
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/radarr:/config
      - /mnt/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped

  # ==========================================
  # RADARR 4K
  # ==========================================
  radarr4k:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr4k
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/radarr4k:/config
      - /mnt/data:/data
    ports:
      - 7879:7878
    restart: unless-stopped

  # ==========================================
  # SEERR
  # ==========================================
  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    user: "13000:13000"
    environment:
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/seerr:/app/config
    ports:
      - 5055:5055
    restart: unless-stopped

  # ==========================================
  # NZBGET
  # ==========================================
  nzbget:
    image: lscr.io/linuxserver/nzbget:latest
    container_name: nzbget
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
    volumes:
      - ~/docker/config/nzbget:/config
      - /mnt/data:/data
      - ~/docker/intermediate_downloads:/data/downloads/intermediate
    ports:
      - 6789:6789
    restart: unless-stopped

  # ==========================================
  # UNPACKERR
  # ==========================================
  unpackerr:
    image: golift/unpackerr:latest
    container_name: unpackerr
    environment:
      - PUID=13000
      - PGID=13000
      - TZ=Europe/Zurich
      - UNPACKERR_INCOMPLETE_DIR=/data/downloads/intermediate

      # --- SONARR HD (Index 0) ---
      - UN_SONARR_0_URL=http://sonarr:8989
      - UN_SONARR_0_API_KEY=<your-api-key>

      # --- SONARR 4K (Index 1) ---
      - UN_SONARR_1_URL=http://sonarr4k:8989
      - UN_SONARR_1_API_KEY=<your-api-key>

      # --- RADARR HD (Index 0) ---
      - UN_RADARR_0_URL=http://radarr:7878
      - UN_RADARR_0_API_KEY=<your-api-key>

      # --- RADARR 4K (Index 1) ---
      - UN_RADARR_1_URL=http://radarr4k:7878
      - UN_RADARR_1_API_KEY=<your-api-key>

      - UNPACKERR_DELETE_AFTER_EXTRACT=true
      - UNPACKERR_WEB_UI_ENABLED=true
      - UNPACKERR_WEB_UI_PORT=5001

    volumes:
      - ~/docker/config/unpackerr:/config
      - /mnt/data:/data
      - ~/docker/intermediate_downloads:/data/downloads/intermediate
    restart: unless-stopped
```

## Hardware-Transcoding (Intel QuickSync)

Da wir einen Intel i5-12600K (12. Generation) verwenden, müssen wir
sicherstellen, dass der Container auf die iGPU zugreifen kann.

### Existenz des Geräts überprüfen

Prüfe, ob das Rendering-Gerät auf dem Host existiert:

```bash
ls -l /dev/dri
```

Du solltest `card0` und `renderD128` sehen.

### Berechtigungen

Der `plex`-Container benötigt die Berechtigung, auf dieses Gerät zuzugreifen. Da
wir das Gerät über Docker durchreichen, wird dies normalerweise automatisch
gehandhabt, aber manchmal muss der Benutzer innerhalb des Containers Teil der
`video`- oder `render`-Gruppe sein.
*Hinweis: Die LinuxServer.io-Images handhaben dies normalerweise automatisch
über das Device-Mapping.*

## Den Stack verwalten

Sobald die Datei gespeichert ist, steuern wir die Dienste mit den üblichen
Docker-Compose-Befehlen.

### Dienste starten

Um Images herunterzuladen und alles im Hintergrund zu starten (Detached-Modus):

```bash
cd ~/docker 
docker compose up -d
```

### Dienste aktualisieren

Um alle Container auf die neueste Version zu aktualisieren:

```bash

cd ~/docker 
# 1. Neueste Images holen
docker compose pull

# 2. Container neu erstellen (aktualisiert nur geänderte) 
docker compose up -d

# 3. Alte, ungenutzte Images entfernen 
docker image prune -f
```

### Logs ansehen

Wenn ein Dienst nicht startet, prüfe die Logs:

```bash
# Logs für einen bestimmten Dienst ansehen 
docker compose logs -f plex

# Logs für alle Dienste ansehen (Strg+C zum Beenden) 
docker compose logs -f
```

## CasaOS

Der Einfachheit halber können wir [CasaOS](https://github.com/IceWhaleTech/CasaOS) einrichten. Das ist sehr einfach und
auf ihrem GitHub gut beschrieben.
