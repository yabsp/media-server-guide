# VPS-Gateway

## VPS-Grundkonfiguration

Wir verwenden einen schlanken VPS (1 vCPU, 2 GB RAM) mit Debian. Wir nutzen die
Hardware-Firewall des Hosting-Anbieters anstelle einer lokalen Firewall (UFW),
da dies einfacher ist und weniger Möglichkeiten für Fehlkonfigurationen lässt.

### IP-Weiterleitung unter Linux

Standardmässig agiert der Linux-Kernel strikt als Host, nicht als Router. Das
bedeutet, dass er jedes Netzwerkpaket, das nicht ausdrücklich an seine eigene
IP-Adresse gerichtet ist, sofort verwirft.

Für unser VPN-Gateway-Setup ist dieses Verhalten ein Hindernis. Der VPS muss als
Brücke fungieren:

-   Er empfängt Verkehr aus dem Internet (z. B. eine Anfrage für Plex).

-   Er muss diesen Verkehr von der physischen Netzwerkkarte (`eth0`) an die
    virtuelle VPN-Schnittstelle (`wg0`) weiterleiten.

Das Aktivieren von `ip_forward` weist den Kernel an, Pakete zwischen
Netzwerkschnittstellen passieren zu lassen. Wir verwenden das
`sysctl.d`-Verzeichnis, damit die Konfiguration dauerhaft erhalten bleibt.

### Server-Einrichtung

```bash
# 1. Grundlagen aktualisieren und installieren 
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard curl git

# 2. IP-Weiterleitung aktivieren
# Wir erstellen eine eigene Konfigurationsdatei in /etc/sysctl.d/, statt die Hauptdatei zu bearbeiten.
# Das ist auf minimalen Linux-Images robuster. 
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf

# Die Änderungen sofort anwenden 
sysctl --system
```

## WireGuard-Konfiguration

Wir verwenden WireGuard, um unseren Media-Server mit unserem VPS zu verbinden. Es
gibt einige Anleitungen, die das Einrichten des Ganzen erleichtern – zum Beispiel
[eine für Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04), die sich auch für andere Versionen und Distributionen
verwenden lässt.
WireGuard verwendet asymmetrische Kryptografie. Wir müssen sowohl auf dem VPS als
auch auf dem Heimserver Schlüsselpaare (privat/öffentlich) erzeugen.

```bash
# Führe dies auf BEIDEN Servern aus, um Schlüssel zu erzeugen: 
wg genkey | tee privatekey | wg pubkey > publickey

# Schlüssel für die Konfiguration anzeigen (Kopiere diese sicher!) 
cat privatekey
cat publickey
```

### VPS-Server-Konfiguration (/etc/wireguard/wg0.conf)

Auf dem VPS konfigurieren wir die Schnittstelle und definieren den Heimserver als
Peer.

```conf
[Interface]
# Internal Tunnel IP of the VPS
Address = 10.10.10.1/24
# Custom Port defined in Firewall 
ListenPort = 51821 
PrivateKey = <VPS_PRIVATE_KEY>

# IP Masquerading Rules (NAT)
# These rules ensure traffic leaving the VPS interface (eth0) appears to come from the VPS IP, \
# enabling the return path for traffic originating from the Home Server. 
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# The Home Server 
PublicKey = <HOME_PUBLIC_KEY> 
# The single IP allowed for the Home Server 
AllowedIPs = 10.10.10.2/32
```

```bash
# Den WireGuard-Dienst auf dem VPS starten 
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

## Client-Konfiguration des Heimservers

Der Heimserver agiert als Client. Er initiiert die Verbindung zum VPS und
verwendet `PersistentKeepalive`, um die NAT-Sitzung durch den Heimrouter offen
zu halten.

```bash
# 1. WireGuard auf dem Heimserver installieren 
sudo apt update && sudo apt install wireguard -y

# 2. Die Schnittstelle konfigurieren 
sudo vim /etc/wireguard/wg0.conf
```

```conf
[Interface] 
# The Internal IP of the Home Server
Address = 10.10.10.2/24 
PrivateKey = <HOME_PRIVATE_KEY>

[Peer] # Connection details for the VPS 
PublicKey = <VPS_PUBLIC_KEY> 
# The Public IP of the VPS and the custom port defined earlier 
# Use this port or any other port you defined (keep in mind that a random port is better)
Endpoint = <VPS_PUBLIC_IP>:51821 
# Route only traffic destined for the VPS internal IP through the tunnel 
AllowedIPs = 10.10.10.1/32 
# Critical: Sends a keepalive packet every 25s to prevent the home router 
# from closing the connection due to inactivity.
PersistentKeepalive = 25
```

```bash
# 3. Den Tunnel starten 
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# 4. Überprüfung 
# Die interne VPS-IP anpingen, um die Verbindung zu bestätigen
ping 10.10.10.1
```

## Reverse-Proxy-Einrichtung (VPS)

Mit aktivem Tunnel stellen wir [Nginx Proxy Manager (NPM)](https://nginxproxymanager.com/guide/) auf dem VPS bereit.
Weitere Informationen findest du in der NPM-Dokumentation. NPM terminiert die
SSL-Verbindungen aus dem Internet und leitet den Verkehr durch den
WireGuard-Tunnel zum Heimserver.

```bash
# 1. Docker auf dem VPS installieren 
curl -fsSL https://get.docker.com | sh

# 2. Verzeichnis und Konfiguration erstellen 
mkdir -p ~/nginx-proxy && cd ~/nginx-proxy 
vim docker-compose.yml
```

```conf
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'     # Public HTTP Port
      - '443:443'   # Public HTTPS Port
      - '81:81'     # Admin Web Interface (setup only!)
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      DISABLE_IPV6: 'true'
    depends_on:
      - crowdsec
```

```bash
# 3. Den Proxy starten 
docker compose up -d
```

### Konfigurationsschritte

1.  **DNS-Eintrag:** Richte einen A-Record (z. B. `plex.yourdomain.com`) auf die öffentliche IP des VPS.

2.  **Admin-UI aufrufen:** Navigiere zu `http://<VPS_IP>:81` (Standard: admin@example.com /
    changeme).

3.  **Proxy-Host erstellen:**

    -   **Domain Name:** plex.yourdomain.com

    -   **Scheme:** http

    -   **Forward Host:** 10.10.10.2 (VPN-IP des Heimservers)

    -   **Forward Port:** 32400

    -   **Websockets:** Aktiviert (entscheidend für Plex)

4.  **SSL-Tab:** Fordere ein neues Let's-Encrypt-Zertifikat an und aktiviere
    "Force SSL".

## Plex-Konfiguration (Heim)

Zum Schluss weisen wir Plex an, die neue öffentliche URL zu verwenden.

1.  Öffne Plex-Einstellungen > Netzwerk.

2.  Setze **Benutzerdefinierte Server-Zugriffs-URLs** auf: `https://plex.yourdomain.com:443`

3.  Deaktiviere und reaktiviere den Fernzugriff, um eine Aktualisierung der Konfiguration zu erzwingen.

## Sicherheitshärtung

Sobald der Proxy-Host konfiguriert ist und bestätigt funktioniert, müssen wir den
VPS absichern, indem wir den Administrationsport schliessen.

Wir wechseln die SSL-Validierungsmethode auf "DNS Challenge". Das würde es uns
erlauben, Port 80 vollständig zu schliessen, sodass nur der verschlüsselte
HTTPS-Port offen bleibt.
*Hinweis:* Wir verwenden hier Cloudflare.

### Cloudflare-API-Token

1. Gehe zum Cloudflare-Dashboard > My Profile > API Tokens.

2. Erstelle ein Token mit der Vorlage **Edit zone DNS** für deine spezifische Domain.

3. Kopiere das erzeugte Token.

### NPM für DNS-Challenge konfigurieren
Bearbeite im Nginx Proxy Manager den Proxy-Host:

- **SSL-Tab:** Aktiviere "Use a DNS Challenge".

- **Provider:** Cloudflare.

- **Zugangsdaten:** Ersetze den Token-Platzhalter:

    ```conf
    dns_cloudflare_api_token = <YOUR_CLOUDFLARE_TOKEN>
    ```

### Finale Firewall-Regeln

Mit aktiver DNS-Validierung können wir Port 80 schliessen. Die einzigen offenen
eingehenden Ports sind nun der VPN-Tunnel und der verschlüsselte HTTPS-Stream.

| Port  | Protokoll | Status | Grund                          |
|------|----------|--------|--------------------------------|
| 22   | TCP      | Erlauben  | SSH-Verwaltung              |
| 443  | TCP      | Erlauben  | HTTPS-Streaming             |
| 51821| UDP      | Erlauben  | WireGuard-Tunnel            |
| 80   | TCP      | VERWEIGERN   | Nicht benötigt (DNS-01 verwendet) |
| 81   | TCP      | VERWEIGERN   | Admin-UI (nur Tunnel)     |

### Sicherer Admin-Zugriff über SSH-Tunnel

Um in Zukunft auf die Nginx-Proxy-Manager-Oberfläche zuzugreifen, ohne den Port
im Internet zu exponieren, verwenden wir lokale Portweiterleitung.

```bash
# Führe dies auf deinem lokalen Rechner aus, um einen sicheren Tunnel zu erstellen
# Syntax: ssh -L <LocalPort>:127.0.0.1:<RemotePort> <User>@<Host> 
ssh -L 8888:127.0.0.1:81 debian@<VPS_PUBLIC_IP>
```

Öffne nun deinen Browser und navigiere zu: **http://localhost:8888**

Der Verkehr ist jetzt innerhalb der SSH-Sitzung verschlüsselt, was ihn selbst
über öffentliches WLAN sicher macht.

## Fortgeschrittene Sicherheit: CrowdSec (IPS)

Da wir den Proxy von Cloudflare (Orange Cloud) umgehen, um direktes Streaming zu
ermöglichen, ist die IP-Adresse des VPS im öffentlichen Internet exponiert. Um
den Server gegen Scanner, Brute-Force-Angriffe und Botnetze zu schützen, setzen
wir **CrowdSec** ein.

### Was ist CrowdSec?

[CrowdSec](https://docs.crowdsec.net/u/getting_started/installation/linux/) ist ein modernes, quelloffenes Intrusion-Prevention-System
(IPS). Anders als ältere Werkzeuge wie Fail2Ban arbeitet es nach einem
kollaborativen Modell:

1. **Erkennen (der Agent):** Er analysiert Logdateien (Nginx, SSH, System) in Echtzeit, um aggressives Verhalten zu erkennen.

2. **Schützen (der Bouncer):** Sobald eine Bedrohung erkannt wird, interagiert der "Bouncer" mit der Firewall, um die Verbindung sofort zu unterbinden.

3. **Teilen (die Crowd):** Wenn dein Server eine IP wegen bösartiger Aktivität blockiert, wird diese Information mit der globalen CrowdSec-Community geteilt. Umgekehrt erhält dein Server automatisch eine Sperrliste von IPs, die andere Nutzer angegriffen haben, und blockiert sie, bevor sie deinen Server überhaupt erreichen.

### Architektureinrichtung
Wir verwenden ein hybrides Setup:

- **Der Agent** läuft in einem **Docker-Container**. Er liest die Logs.

- **Der Bouncer** läuft direkt auf dem **Host-System (VPS)**. Er verwaltet die `iptables`-Firewall-Regeln.

### 1. Docker-Compose-Anpassung (der Agent)
Wir passen die `docker-compose.yml` an, um CrowdSec hinzuzufügen. **Wichtige Änderungen:**

- Wir mappen die `auth.log` des Hosts, um SSH zu schützen.

- Wir exponieren Port `8080` gegenüber localhost, damit der Host-Bouncer mit dem Docker-Agenten kommunizieren kann.

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # Public HTTP Port
      - '80:80'
      # Public HTTPS Port
      - '443:443'
      # Admin Web Interface (setup only!)
      - '81:81'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      DISABLE_IPV6: 'true'
    depends_on:
      - crowdsec

  crowdsec:
    image: crowdsecurity/crowdsec:latest
    restart: unless-stopped
    environment:
      # load Nginx and SSHD rules
      - COLLECTIONS=crowdsecurity/nginx-proxy-manager crowdsecurity/sshd
    volumes:
      # Nginx logs
      - ./data/logs:/var/log/npm-logs:ro
      # ssh logs of host to container
      - /var/log/auth.log:/var/log/host-auth.log:ro
      # CrowdSec data
      - ./crowdsec-db:/var/lib/crowdsec/data
      - ./crowdsec-config:/etc/crowdsec
    ports:
      - 8080:8080
    security_opt:
      - no-new-privileges=true

```

### 2. Konfiguration der Log-Erfassung

CrowdSec muss genau wissen, welche Dateien es analysieren soll. Wir erstellen
`crowdsec-config/acquis.yaml`, das die Nginx-Logs und die
System-Authentifizierungslogs zuordnet.

```conf
# --- Nginx proxy manager logs ---
filenames:
  - /var/log/npm-logs/proxy-host-*_access.log
  - /var/log/npm-logs/proxy-host-*_error.log
  - /var/log/npm-logs/fallback_access.log
  - /var/log/npm-logs/fallback_error.log
labels:
  type: nginx-proxy-manager

# --- SSH host logs ---
filenames:
  - /var/log/host-auth.log
labels:
  type: syslog

```

Wende die Änderungen an, indem du den Stack neu startest:

```bash
docker compose up -d 
docker compose restart crowdsec
```

### 3. Den Enforcer installieren (Firewall-Bouncer)

Der Docker-Container erkennt die Angriffe, kann aber die Firewall des Hosts nicht
direkt steuern. Wir installieren den `iptables-bouncer` auf dem VPS-Host-System.

```bash
# 1. Repository hinzufügen und installieren 
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
sudo apt install crowdsec-firewall-bouncer-iptables -y

# 2. API-Schlüssel erzeugen (innerhalb von Docker) 
# Der Bouncer benötigt diesen Schlüssel, um sich beim Docker-Agenten zu authentifizieren 
docker exec -t nginx-proxy-crowdsec-1 cscli bouncers add firewall-bouncer 
# > Kopiere den hier erzeugten API-SCHLÜSSEL!

# 3. Bouncer konfigurieren 
sudo vim /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

Bearbeite die Konfigurationsdatei, damit sie auf den lokalen Docker-Port zeigt:

```conf
mode: iptables
update_frequency: 10s
log_mode: file
log_dir: /var/log/
log_level: info
# Point to localhost port 8080 (exposed by Docker)
api_url: http://127.0.0.1:8080/
# Paste your generated key here
api_key: <YOUR_GENERATED_KEY>
disable_ipv6: true
```

```bash
# 4. Schutz aktivieren 
sudo systemctl restart crowdsec-firewall-bouncer

# 5. Status überprüfen
# Prüfen, ob der Bouncer im Container registriert ist 
docker exec -t nginx-proxy-crowdsec-1 cscli bouncers list
```

### Registrierung & Überprüfung

Standardmässig hat eine frische CrowdSec-Installation eine leere Sperrliste. Um
sofort die vollständige "Community Blocklist" (ca. 20'000+ bösartige IPs) zu
erhalten und Angriffe zu visualisieren, registrieren wir die Instanz in der
CrowdSec-Konsole.

1. Gehe zu [https://app.crowdsec.net](https://app.crowdsec.net) und erstelle ein kostenloses Konto.

2. Wähle "Enroll my first instance", um einen Registrierungsschlüssel zu erhalten.

3. Führe den Registrierungsbefehl innerhalb des Docker-Containers aus:

```bash
# Ersetze <KEY> durch den Code von der Website 
docker exec -t nginx-proxy-crowdsec-1 cscli console enroll <KEY>

# Neustart, um ein sofortiges Update der Sperrliste zu erzwingen
docker compose restart crowdsec
```

**Überprüfung:** Das Herunterladen der globalen Sperrliste (CAPI) dauert etwa
5–10 Minuten. Überprüfe danach, ob sich dein Server gegen Tausende bekannter
Übeltäter geschützt hat:

```bash
# Die Anzahl der gesperrten IPs aus der Community-Liste prüfen 
docker exec -t nginx-proxy-crowdsec-1 cscli decisions list --origin CAPI -a | grep -v "No decisions" | wc -l
```

*Erwartetes Ergebnis:* Eine Zahl grösser als **15'000**. Wenn das Ergebnis 0
oder 1 ist, warte noch ein paar Minuten und versuche es erneut.
