# Homelab Guide

Diese Seite sammelt die Anleitungen zu meinem Heimserver und dem
Netzwerk darum herum. Sie besteht aus zwei voneinander unabhängigen
Teilen; du kannst jeden für sich lesen.

| Teil | Inhalt | Für wen |
| --- | --- | --- |
| [Medienserver](#medienserver) | Debian-Server mit Plex und den *arr-Apps | Wer den Server selbst aufbaut |
| [Heimnetzwerk](#heimnetzwerk) | WLAN mit UniFi Access Points | Wer die Access Points betreibt oder neu einrichtet |

## Medienserver

Einen richtigen Linux-Server aufzusetzen, um eine Mediensammlung zu hosten, kann ziemlich zeitaufwändig sein – selbst mit modernen Werkzeugen. Deshalb habe ich diese Anleitung erstellt. Sie beginnt bei der [Debian-Installation](01-debian.md).

### Wie ist die Anleitung zu lesen?

Dieser Teil ist in Kapitel und Abschnitte gegliedert. Die Kapitel sind geordnet, und die Reihenfolge der Kapitel sowie der Abschnitte innerhalb jedes Kapitels entspricht genau der Reihenfolge, in der die Schritte für die Einrichtung ausgeführt werden sollten.

### Arr-Apps und Weiteres

Detaillierte Konfigurationsanleitungen für Sonarr, Radarr, Seerr, NZBGet, Unpackerr und Plex sind noch nicht enthalten. Da die Vorlieben der Nutzer bei diesen Werkzeugen unterschiedlich sind, haben diese Anleitungen eher tiefe Priorität – nichtsdestotrotz sind sie für zukünftige Aktualisierungen geplant. Weiterführendes Material findest du unter [Weiterführende Links & verwandte Anleitungen](09-further-reading.md).

### Rechtlicher Hinweis

Dieser Hinweis betrifft nur den Medienserver-Teil. Bitte beachte, dass das Herunterladen von urheberrechtlich geschütztem Material nicht in jeder Region legal ist. Die Anleitung zum Medienserver ist spezifisch so geschrieben, dass sie dem [Bundesgesetz über das Urheberrecht und verwandte Schutzrechte](https://www.fedlex.admin.ch/eli/cc/1993/1798_1798_1798/de) der Schweiz folgt.

## Heimnetzwerk

Das WLAN im Haus läuft über zwei Ubiquiti UniFi Access Points. Dieser Teil ist für den Medienserver nicht nötig und besteht aus zwei Seiten für unterschiedliche Leser:

- [UniFi Access Points](12-unifi-access-points.md) – alles zu den Geräten selbst: Was die Lampe bedeutet, wie man nachsieht, ob alles läuft, was bei einer Störung zu tun ist und wie man ein Gerät zurücksetzt und neu einrichtet. Das ist die Seite für den Alltag.
- [UniFi Controller (Docker)](11-unifi-controller.md) – die einmalige Installation des Controllers auf dem Debian-Server: Docker Compose, Ports, Datenbank. Wird nur gebraucht, wenn der Controller neu aufgesetzt wird.

## In Arbeit

Diese Dokumentation ist keineswegs endgültig und kann jederzeit weiter verbessert werden.
Die Sicherheit wurde berücksichtigt, dennoch kann es weiterhin Risiken und Probleme mit diesem Setup geben. Es wird erwartet, dass du dir selbst Gedanken darüber machst, warum wir etwas tun und wie man es richtig macht.
