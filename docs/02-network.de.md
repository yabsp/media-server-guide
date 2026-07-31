# Grundkonfiguration

## Benutzerverwaltung

Da wir das Root-Passwort übersprungen haben, verfügt unser erstellter Benutzer
(nennen wir ihn `joachim` oder ähnlich) über Sudo-Rechte.

### Sudo-Zugriff überprüfen

Um zu überprüfen, ob der Benutzer administrative Aufgaben ausführen kann:

```bash
# Prüfen, ob der Benutzer zur sudo-Gruppe gehört 
groups

# Sudo-Zugriff testen (sollte nach dem Benutzerpasswort fragen) 
sudo apt update
```

### Hostnamen setzen

Damit der Server im Netzwerk leicht identifizierbar ist:

```bash
sudo hostnamectl set-hostname mediaserver
```

## Einrichtung eines dedizierten Dienstbenutzers

Aus Sicherheitsgründen und für eine saubere Rechteverwaltung führen wir die
Anwendungen nicht als administrativer Benutzer aus. Stattdessen erstellen wir
einen dedizierten Systembenutzer (namens `media`) für alle Docker-Container.

### Den Media-Benutzer erstellen

Wir erstellen einen Systembenutzer mit einer bestimmten UID und deaktivieren aus
Sicherheitsgründen die Anmeldemöglichkeit.

```bash
# Benutzer 'media' mit UID 13000 erstellen, passende Gruppe, kein Login, kein Home
sudo useradd -u 13000 -U -M -s /bin/false media

# Erstellung überprüfen
id media
# Ausgabe: uid=13000(media) gid=13000(media) groups=13000(media)
```

- **PUID:** 13000

- **PGID:** 13000

- `-U`: Erstellt eine Gruppe mit demselben Namen und derselben ID.

- `-M`: Kein Home-Verzeichnis erstellen.

- `-s /bin/false`: Anmeldung deaktivieren.

### Dem Admin-Benutzer Zugriff gewähren

Damit wir (der angemeldete Admin) weiterhin Dateien im Speicherpool manuell
verwalten können, fügen wir unseren aktuellen Benutzer der neuen Gruppe hinzu.

```bash
sudo usermod -a -G media joachim
```

### Dateisystem-Berechtigungen anwenden

Wir übertragen die Eigentümerschaft des Datenverzeichnisses auf den neuen
Benutzer und die neue Gruppe. Wir setzen die Berechtigungen auf `775`, was
sowohl dem Dienstbenutzer als auch den Gruppenmitgliedern (uns) vollen Zugriff
gewährt.

```bash
# Eigentümerschaft auf media:media ändern 
sudo chown -R media:media /mnt/data

# Berechtigungen setzen (User=RWX, Group=RWX, Others=RX) 
sudo chmod -R 775 /mnt/data/
```

*NB: r = 4, w = 2, x = 1*

### Nur-Lese-Zugriff für Plex erzwingen

Während Dienste wie Sonarr Schreibzugriff benötigen, um Dateien zu verwalten,
sollte Plex keine Mediendateien verändern.

Anstatt komplexer Dateisystem-ACLs regeln wir dies später auf Docker-Ebene.

- Alle Container laufen als `PUID=13000`.

- Sonarr/Radarr binden Volumes als **Lese-Schreib** ein.

- Plex bindet das Medien-Volume mit dem `:ro`-Flag (Read-Only) in Docker
    Compose ein (siehe [Docker & Dienste](04-docker.md)).

## Netzwerkkonfiguration

Da es sich um einen Server handelt, benötigt er eine statische IP. In meinem
Fall habe ich die folgende Schnittstelle hinzugefügt:

```bash
# Die Datei /etc/network/interfaces bearbeiten
sudo vim /etc/network/interfaces
```
Und füge hinzu:

```conf
# The primary network interface
allow-hotplug enp4s0
iface enp4s0 inet static
	address 192.168.0.50
	netmask 255.255.255.0
	gateway 192.168.0.1
	dns-nameservers 1.1.1.1 8.8.8.8
```

- `allow-hotplug`: Weist das System an, die Netzwerkschnittstelle `enp4s0` zu
    starten, sobald der Kernel erkennt, dass die Hardware vorhanden ist. Das
    verhindert, dass der Bootvorgang beim Warten auf die Schnittstelle hängen
    bleibt, wie es mit `auto` passieren kann.

- `inet`: IPv4 verwenden (für IPv6 `inet6` verwenden).

- `gateway`: IP-Adresse des Routers.

Folgende Befehle sind nützlich:

```bash
# 1. Den Schnittstellennamen finden (z. B. eth0, enp4s0, eno1) 
ip a

# 2. Die Gateway-IP finden (achte auf die Zeile 'default via ...') 
ip r
```

## Sprache & Gebietsschema

Wir haben die Systemsprache auf Englisch gesetzt, aber das Tastaturlayout auf
Schweizerdeutsch (oder Deutsch) belassen, da ich eine Schweizer Tastatur
verwende.

```bash
# Locales neu konfigurieren (auf en_US.UTF-8 setzen)
sudo dpkg-reconfigure locales

# Tastaturlayout prüfen (sollte 'ch' oder 'de' sein) 
cat /etc/default/keyboard
```

## SSH-Server-Installation

Falls der SSH-Server bei der ersten Betriebssysteminstallation nicht ausgewählt
wurde, muss er manuell installiert werden.

### Installation über APT

```bash
# 1. Das Paket installieren 
sudo apt update 
sudo apt install openssh-server

# 2. Den Dienst aktivieren und starten 
sudo systemctl enable --now ssh

# 3. Den Status überprüfen 
sudo systemctl status ssh
```

### Verbindung zum Server herstellen

Von einem entfernten Rechner (Windows PowerShell oder Linux-Terminal):

```bash
# Syntax: ssh benutzername@ip-adresse 
ssh user@192.168.0.50
```

## SSH-Schlüssel-Authentifizierung (passwortlos)

Um Sicherheit und Komfort zu erhöhen, konfigurieren wir die
SSH-Schlüssel-Authentifizierung. Diese erlaubt die Anmeldung ohne Eingabe eines
Passworts, indem stattdessen ein kryptografisches Schlüsselpaar verwendet wird.

### 1. Ein Schlüsselpaar erzeugen (auf dem Client)

**Hinweis:** Diese Befehle sind auf deinem **Laptop/PC** auszuführen. Wir
verwenden den `ed25519`-Algorithmus.

```bash
# Führe dies auf deinem lokalen Rechner aus 
ssh-keygen -t ed25519 -C "my-laptop"

# Drücke Enter, um den Standard-Speicherort zu übernehmen.
```

### 2. Den öffentlichen Schlüssel auf den Server kopieren

Wir übertragen den öffentlichen Teil des Schlüssels mit `ssh-copy-id` auf den
Server.

```bash
# Ersetze Benutzer und IP durch deine Serverdaten 
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.0.50
```

### 3. Überprüfen und Härten

Versuche, dich erneut anzumelden. Du solltest nicht nach einem Passwort gefragt
werden. Sobald dies bestätigt ist, deaktivieren wir die Passwort-Authentifizierung,
um Brute-Force-Angriffe zu verhindern.

```bash
# Öffne die SSH-Konfiguration auf dem SERVER 
sudo vim /etc/ssh/sshd_config
```

Suche und ändere die folgenden Zeilen:

```conf
# Disable root login (Security Best Practice)
PermitRootLogin no

# Disable password authentication 
PasswordAuthentication no
ChallengeResponseAuthentication no 
UsePAM yes
```

Starte SSH neu, um die Änderungen anzuwenden:

```bash
sudo systemctl restart ssh
```

## Firewall-Einrichtung (UFW)

Um das Host-System abzusichern, verwenden wir **UFW** (Uncomplicated Firewall).
Obwohl Docker seine eigenen Netzwerkregeln tief im System verwaltet (iptables),
ist UFW unverzichtbar, um die Host-Dienste (wie SSH) zu schützen, und dient als
zweite Verteidigungsebene.

### Installation und Standardrichtlinien

Zuerst installieren wir UFW und definieren das grundlegende Verhalten:
standardmässig sämtlichen eingehenden Verkehr blockieren, aber allen
ausgehenden Verkehr erlauben (damit der Server Updates herunterladen kann).

```bash
# UFW installieren sudo apt update 
sudo apt install ufw

# Standards setzen (noch nicht aktivieren) 
sudo ufw default deny incoming 
sudo ufw default allow outgoing
```

### Regeln definieren

Wir unterscheiden zwischen kritischem Systemzugriff (SSH) und Anwendungszugriff
(Plex, Weboberflächen).

#### 1. SSH absichern (entscheidend)

Wir beschränken den SSH-Zugriff strikt auf das lokale Heimnetzwerk (LAN). Das
verhindert, dass jemand aus dem offenen Internet überhaupt versucht, sich
anzumelden – selbst wenn er den Schlüssel hätte.

**Hinweis:** Ersetze `192.168.0.0/24` durch dein tatsächliches Subnetz.

```bash
# SSH nur aus dem lokalen LAN erlauben 
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
```

#### 2. Anwendungs-Ports (Vorkonfiguration)

Obwohl wir Docker noch nicht installiert haben, konfigurieren wir die Firewall
für die geplanten Dienste vor.

**Öffentlicher Zugriff (Internet & LAN):**

```bash
# Plex (Streaming) - Standard-Port 32400 
sudo ufw allow 32400/tcp
```

**Privater Admin-Zugriff (nur LAN):** Verwaltungsoberflächen wie Sonarr,
Radarr oder Portainer sollten nicht im offenen Internet exponiert werden. Wir
beschränken sie auf das lokale Netzwerk.

```bash
# CasaOS / Dashboard (Port 80), nur falls du dies verwenden möchtest
sudo ufw allow from 192.168.0.0/24 to any port 80 proto tcp

# Overseerr/Seerr (Anfragenverwaltung - Port 5055) 
sudo ufw allow from 192.168.0.0/24 to any port 5055 proto tcp

# Sonarr (8989) & Radarr (7878) 
sudo ufw allow from 192.168.0.0/24 to any port 8989 proto tcp 
sudo ufw allow from 192.168.0.0/24 to any port 7878 proto tcp

# NZBGet / SABnzbd (6789) 
sudo ufw allow from 192.168.0.0/24 to any port 6789 proto tcp
```

**Hinweis**: Da wir diese Werkzeuge mit Docker bereitstellen, benötigen wir
diese UFW-Regeln streng genommen nicht; Docker manipuliert `iptables`
standardmässig direkt und umgeht UFW. Sie sind hier der Vollständigkeit halber
aufgeführt sowie für Dienste, die du direkt auf dem Host betreibst.

### Die Firewall aktivieren

**Warnung:** Stelle sicher, dass die SSH-Regel (Abschnitt [SSH-Server-Installation](02-network.md#ssh-server-installation))
zu deiner aktuellen IP-Adresse passt, sonst sperrst du dich womöglich sofort
selbst aus.

```bash
# UFW aktivieren 
sudo ufw enable

# Status und nummerierte Regeln prüfen
sudo ufw status numbered
```
