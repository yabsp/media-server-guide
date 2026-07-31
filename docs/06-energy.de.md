# Energieoptimierung

Ein Server, der rund um die Uhr läuft, belastet die Stromrechnung. Da ein
Media-Server den grössten Teil des Tages im Leerlauf ist, optimieren wir die
Hardware so, dass sie bei Nichtgebrauch in stromsparende Zustände wechselt.

## Festplatten-Spindown

Mechanische Festplatten verbrauchen im Betrieb etwa 6–10 Watt, im Standby
jedoch weniger als 1 Watt. Da wir **MergerFS** (statt RAID 5/ZFS) verwenden,
haben wir einen grossen Vorteil: Wir können Festplatten einzeln herunterfahren.
Beim Anschauen eines Films muss nur die Festplatte aufwachen, die diese Datei
enthält; die übrigen bleiben im Ruhezustand.

### Konfiguration über hdparm

Wir verwenden `hdparm`, um eine Zeitüberschreitung festzulegen. Ist eine
Festplatte für eine bestimmte Zeit inaktiv, stoppt der Motor.

```bash
sudo apt install hdparm -y
```

Damit die Konfiguration Neustarts übersteht und auch dann funktioniert, wenn
sich die Laufwerksbuchstaben ändern (z. B. /dev/sdb wird zu /dev/sdc),
identifizieren wir die Laufwerke anhand ihrer UUID.

```bash
# 1. UUIDs deiner HDDs finden (die SSD/Boot-Platte ignorieren!) 
lsblk -f

# 2. Konfiguration bearbeiten 
sudo vim /etc/hdparm.conf
```

Füge für **jede** mechanische Festplatte am Ende der Datei einen Block hinzu:

```conf
# HDD 1
/dev/disk/by-uuid/DEINE-UUID-HIER {
    # 241 = 30 Min, 242 = 1 Stunde
    spindown_time = 241
    # Aggressives Energiemanagement
    apm = 127
}
# HDD 2
/dev/disk/by-uuid/NAECHSTE-UUID-HIER {
    spindown_time = 241
    apm = 127
}
```

*Hinweis:* Eine Spindown-Zeit von 30–60 Minuten wird empfohlen. Ein zu kurzer
Wert (z. B. 5 Minuten) führt zu häufigen Start-Stopp-Zyklen, was die Mechanik
der Festplatte abnutzt.

## Überprüfung

Um zu prüfen, ob deine Festplatten tatsächlich schlafen, ohne sie aufzuwecken:

```bash
sudo hdparm -C /dev/sd?
```

*Ausgabe:*

- `drive state is: active/idle` -> Festplatte dreht sich (verbraucht Strom).

- `drive state is: standby` -> Festplatte schläft (spart Strom).
