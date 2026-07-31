# Speichereinrichtung (MergerFS)

Wir nutzen **MergerFS**, um mehrere physische Festplatten zu einem einzigen
logischen Speicherpool zusammenzufassen. Anders als bei RAID können hier
Festplatten unterschiedlicher Grösse gemischt werden, und die Daten bleiben auf
den einzelnen Festplatten lesbar, falls der Pool ausfällt. Die Geschwindigkeit
ist hier ein kleiner Kompromiss, sollte aber für das, was wir mit unserem Server
vorhaben, kein Problem sein.

## Voraussetzungen

Wir benötigen die nötigen Softwarepakete, um den Pool und verschiedene
Dateisysteme (wie exFAT für die Migration) zu handhaben.

```bash
# MergerFS und Dateisystem-Werkzeuge installieren 
sudo apt update 
sudo apt install mergerfs fuse3 exfatprogs
```

## Festplattenvorbereitung

### Laufwerke identifizieren

Vor dem Formatieren müssen wir die korrekten Gerätebezeichner identifizieren
(z. B. `/dev/sda`, `/dev/sdb`).

```bash
# Blockgeräte und ihre Dateisysteme auflisten 
lsblk -f
```

### Formatieren auf ext4

Linux funktioniert am besten mit dem `ext4`-Dateisystem. Wenn wir ein neues
Laufwerk für die Medienspeicherung vorbereiten, verwenden wir bestimmte Flags,
um den nutzbaren Speicherplatz zu maximieren.

**Warnung:** Dieser Befehl löscht alle Daten auf dem Ziellaufwerk!

```bash
# Das Laufwerk formatieren (sdX durch den tatsächlichen Bezeichner ersetzen)
sudo mkfs.ext4 -m 0 -L disk1 /dev/sdX
```

**Erklärung der Parameter:**

- `-m 0`: Standardmässig reserviert ext4 5 % des Speicherplatzes für den
    Root-Benutzer / für Systemdienste. Bei einem 10-TB-Laufwerk würde das 500 GB
    verschwenden. Da dies ein Datenlaufwerk ist, setzen wir die reservierten
    Blöcke auf 0 %.

- `-L label`: Weist der Partition ein Label (einen Namen) zu, was die spätere
    Identifikation erleichtert.

## Laufwerke einhängen

Wir hängen Laufwerke nicht über ihren Gerätenamen ein (z. B. `/dev/sdb`), da
sich diese Namen nach einem Neustart ändern können. Stattdessen verwenden wir
die **UUID**.

### Die UUID ermitteln

```bash
# UUIDs aller Partitionen anzeigen 
sudo blkid
```

*Kopiere die UUID-Zeichenkette (ohne Anführungszeichen) für den nächsten
Schritt. Wenn du dies in einer reinen Shell-Umgebung machst, aber eine Maus zur
Verfügung hast, möchtest du vielleicht [`gpm`](https://linux.die.net/man/8/gpm)
zum Kopieren und Einfügen verwenden.*

### fstab konfigurieren

Wir bearbeiten `/etc/fstab`, um die Laufwerke beim Booten automatisch
einzuhängen.

```bash
# Zuerst die Einhängepunkte erstellen 
sudo mkdir -p /mnt/disk1 
sudo mkdir -p /mnt/disk2

# Die Dateisystemtabelle bearbeiten 
sudo vim /etc/fstab
```

Füge die folgenden Zeilen für die physischen Festplatten hinzu:

```conf
# Physical Disk 1 
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /mnt/disk1 ext4 defaults 0 2

# Physical Disk 2 
UUID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy /mnt/disk2 ext4 defaults 0 2
```

## Der MergerFS-Pool

Zum Schluss definieren wir den virtuellen Pool, der `/mnt/disk1`, `/mnt/disk2`
usw. zu `/mnt/data` zusammenfasst.

Füge diese Zeile am Ende von `/etc/fstab` hinzu:

```conf
# MergerFS Pool
# Syntax: branches mount_point type options dump pass 
/mnt/disk* /mnt/data fuse.mergerfs defaults,nonempty,allow_other,use_ino,category.create=mfs,minfreespace=10G,fsname=mergerfs 0 0
```

### Die MergerFS-Argumente verstehen

- `/mnt/disk*`: Die Quelle. Der Stern (\*) weist MergerFS an, **jeden** Ordner
    in diesem Verzeichnis einzubeziehen, der mit «disk» beginnt. Das erleichtert
    das spätere Hinzufügen neuer Laufwerke (disk3, disk4).

- `defaults`: Standard-Einhängeeinstellungen.

- `allow_other`: **Entscheidend für Docker.** Erlaubt anderen Benutzern als dem,
    der das Dateisystem eingehängt hat (root), den Zugriff. Ohne dies würde Plex
    die Dateien nicht sehen.

- `use_ino`: Übergibt die ursprünglichen Inode-Nummern der physischen
    Festplatten. Wichtig, damit Hardlinks über den Pool hinweg korrekt
    funktionieren.

- `category.create=mfs`: **«Most Free Space»** (am meisten freier Speicher).
    Wird eine neue Datei in den Pool geschrieben, schreibt MergerFS sie auf die
    physische Festplatte mit dem meisten verfügbaren Speicherplatz. Das hält die
    Laufwerksauslastung ausgeglichen.

- `minfreespace=10G`: Verhindert, dass eine Festplatte zu 100 % voll wird, was
    zu Dateisystemkorruption oder Abstürzen führen kann. Hat eine Festplatte
    weniger als 10 GB übrig, wechselt MergerFS zur nächsten.

## Änderungen anwenden

Sobald `fstab` gespeichert ist, erstelle das Pool-Verzeichnis und hänge alles
ein.

```bash
# Den Pool-Einhängepunkt erstellen 
sudo mkdir -p /mnt/data

# Alle in fstab aufgeführten Dateisysteme einhängen 
sudo mount -a

# Die Pool-Grösse überprüfen (sollte die Summe aus disk1 + disk2 sein) 
df -h /mnt/data
```
