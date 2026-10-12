# UniFi Controller (Docker)

WiFi in the house runs on two Ubiquiti **UAP-AC-PRO** (`UG-AP` and
`Licht-AP`), both connected by network cable. UniFi devices
have no web interface of their own; they are configured through a
central **controller** (UniFi Network Application). We run it as one
more container on the media server.

The controller is only needed for setup, changes and firmware updates.
WiFi keeps working while the server is off.

This page only covers installing the controller on the Debian server.
Everything about the devices themselves (setting up, resetting,
operating) is in [UniFi Access Points](12-unifi-access-points.md).

!!! warning "The standalone controller is being phased out"

    The interface (version 10.0) announces on opening that upcoming
    Network versions will only be released for **UniFi OS Server**. The
    container described here keeps working, but will stop receiving new
    versions at some point. Moving over is possible via backup/restore.

## 1. Controller in Docker Compose

### Ports

| Port | Purpose |
| --- | --- |
| `8443/tcp` | Web interface |
| `8081/tcp` | Inform (access point -> controller) |
| `3478/udp` | STUN |
| `10001/udp` | Device discovery |

The default inform port would be `8080`. On this server it is already
taken by another container, so we use `8081`.

### Database Init Script

The controller needs a MongoDB with a dedicated user. The script creates
it on the very first start of the database.

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

### Services

Add to `~/docker/docker-compose.yml` under `services:`:

```yaml
  # ==========================================
  # UNIFI - Database
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
      - 8443:8443       # Web interface
      - 8081:8081       # Inform
      - 3478:3478/udp   # STUN
      - 10001:10001/udp # Device discovery
    restart: unless-stopped
```

!!! warning "Three pitfalls"

    - **Passwords made of letters and digits only.** Special characters
      such as `:`, `@` or `/` make the database connection URL invalid.
      Generate them with `openssl rand -hex 16`.
    - **No `network_mode: host`.** On the host network the controller
      cannot resolve the name `unifi-db`.
    - **`MONGO_PASS` identical in both places.** The password is only
      picked up on the first start. To change it later,
      `~/docker/config/unifi-db/data` and `~/docker/config/unifi` have
      to be deleted.

### First Start

The database first, then the controller. The inform port is set in the
configuration file, which is only created on the first start.

```bash
cd ~/docker
docker compose up -d unifi-db
docker compose up -d unifi

# Wait about a minute, then set the inform port
docker compose stop unifi
echo "unifi.http.port=8081" | sudo tee -a ~/docker/config/unifi/data/system.properties
docker compose up -d unifi
```

Check from another machine that the inform port answers. An
`HTTP/1.1 400` is the expected reply.

```bash
curl -i http://192.168.0.50:8081/inform
```

The web interface is at `https://192.168.0.50:8443`. The certificate
warning in the browser is normal. Create a local account in the wizard.

Continue with the basic settings and the first device in
[UniFi Access Points](12-unifi-access-points.md#8-basic-controller-settings).

## 2. Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Defined MONGO_HOST unifi-db is not reachable` | `unifi` runs with `network_mode: host` or in a different Compose project than `unifi-db`. |
| `The connection string contains invalid user information` | Special characters in `MONGO_PASS`. Set a new password and delete both data directories. |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Port already in use. Move the inform port to `8081` as above. |
