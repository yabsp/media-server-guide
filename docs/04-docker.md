# Docker & Services

We use **Docker** to containerize applications. This keeps the base OS
clean and allows for easy updates and backups. We orchestrate these
containers using **Docker Compose**, which allows us to define the
entire server configuration in a single text file.

## Installation

We install the official Docker Engine from the Docker repository to
ensure we have the latest version (Debian's default repositories are
often outdated). The whole process is very well documented in the [Docker
documentation](https://docs.docker.com/engine/install/debian/).

### User Permissions

By default, Docker commands require 'sudo'. To fix this, we add our
admin user to the 'docker' group.

## Directory Structure

We keep all Docker configurations in our home directory. This makes
backing up the server configuration easy: just backup this one folder.

```bash
# Create the main docker directory 
mkdir -p ~/docker

# Create a directory for the compose file 
cd ~/docker 
touch docker-compose.yml
```

## The Docker Compose File

This is the heart of the server. We edit the file:

```bash
vim ~/docker/docker-compose.yml
```

Below is the configuration template optimized for:

- **Intel i5-12600K:** QuickSync Transcoding enabled.

- **Service User:** Using UID 13000 as defined in chapter [User & Network](02-network.md)

- **Hardlinks:** Optimized volume mapping for instant moves.

- **Security:** Plex gets Read-Only access.

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

## Hardware Transcoding (Intel QuickSync)

Because we are using an Intel i5-12600K (12th Gen), we need to ensure
the container can access the iGPU.

### Verify Device Existence

Check if the rendering device exists on the host:

```bash
ls -l /dev/dri
```

You should see `card0` and `renderD128`.

### Permissions

The `plex` container needs permission to access this device. Since we pass the
device through Docker, this is usually handled automatically, but
sometimes the user inside the container needs to be part of the `video` or `render` group. 
*Note: The LinuxServer.io images usually handle this
automatically via the device mapping.*

## Managing the Stack

Once the file is saved, we control the services using standard Docker
Compose commands.

### Starting Services

To download images and start everything in the background (detached
mode):

```bash
cd ~/docker 
docker compose up -d
```

### Updating Services

To update all containers to the latest version:

```bash

cd ~/docker 
# 1. Pull latest images
docker compose pull

# 2. Recreate containers (only updates changed ones) 
docker compose up -d

# 3. Remove old unused images 
docker image prune -f
```

### Viewing Logs {#viewing-logs .unnumbered}

If a service isn't starting, check the logs:

```bash
# View logs for a specific service 
docker compose logs -f plex

# View logs for all services (Press Ctrl+C to exit) 
docker compose logs -f
```

## CasaOS

For convenience purposes we may set up [CasaOS](https://github.com/IceWhaleTech/CasaOS). This is very easy and
properly described on their github.

