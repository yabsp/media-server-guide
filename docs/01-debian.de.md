# Basissystem-Installation

## Einrichtung des Betriebssystems

### Installationsphilosophie

Das Ziel ist ein **Headless-Server**, um den Ressourcenverbrauch zu minimieren.

- **Betriebssystem:** **Debian 13 (Trixie)**, aber grundsätzlich eignet sich hier jedes Linux-Betriebssystem.
- **GUI:** Keine (kein GNOME/KDE). Nur Terminal. Optional kannst du eine Desktop-Umgebung (z. B. GNOME) installieren, sie aber deaktiviert lassen, sofern du sie nicht wirklich brauchst.

    ```bash
    sudo systemctl set-default multi-user.target
    ```

- **Dateisystem:** ext4.

### Strategie für das «Root»-Passwort

Während des Installationsprozesses fragt Debian nach einem `root`-Passwort.

- **Vorgehen:** Wir lassen das Feld für das Root-Passwort leer.
- **Begründung:** Dadurch deaktiviert Debian die direkte Anmeldung über das Root-Konto und fügt den primären Benutzer automatisch der `sudo`-Gruppe hinzu. Das ist einfacher und wohl auch sicherer, als einen Benutzer mit demselben Passwort wie Root zu haben.

### Softwareauswahl

Im *tasksel*-Menü (blauer Bildschirm gegen Ende) wählen wir:

1. SSH-Server (entscheidend für den Fernzugriff); falls du dies nicht ausgewählt hast, findest du unter [Benutzer & Netzwerk](02-network.md#ssh-server-installation) die Installation des SSH-Servers.
2. Standard-Systemwerkzeuge
