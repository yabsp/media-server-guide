# Weiterführende Links & verwandte Anleitungen

Dieses Kapitel sammelt die externen Ressourcen, Anleitungen und Referenzen, die
dieses Setup beeinflusst haben. Sie sind ein guter Ausgangspunkt, wenn du tiefer
in ein bestimmtes Thema eintauchen oder von den in dieser Anleitung getroffenen
Entscheidungen abweichen möchtest.

## Einrichtungsanleitungen & Plattformen

Umfassendere Anleitungen zu vollständigen Media-Server-Stacks und alternativen
Plattformen.

- [Docker Media Server (SmartHomeBeginner)](https://www.smarthomebeginner.com/docker-media-server-2022/)
    — Eine umfassende Schritt-für-Schritt-Anleitung für einen Docker-basierten
    Media-Server mit über 20 Apps.
- [Detailed Guide to Setting Up ZFS RAID on Ubuntu](https://manishrjain.com/zfs-raid-ubuntu)
    — Ein alternativer Speicheransatz zum MergerFS-Pool aus der
    [Speichereinrichtung](03-storage.md).
- [CasaOS](https://github.com/IceWhaleTech/CasaOS)
    — Ein einfaches, elegantes quelloffenes Personal-Cloud-System, falls du eine
    schlüsselfertige Oberfläche einem selbstgebauten Stack vorziehst.

## Arr-Apps & Automatisierung

Referenzmaterial für das *Arr*-Ökosystem und die Werkzeuge, die es
zusammenhalten.

- [TRaSH Guides — Collection of Custom Formats (Radarr)](https://trash-guides.info/Radarr/Radarr-collection-of-custom-formats/)
    — Der De-facto-Standard für Qualitätsprofile und Custom Formats.
- [TRaSH Guides — How to set up Language Custom Formats](https://trash-guides.info/Sonarr/Tips/How-to-setup-language-custom-formats/)
    — Nützlich, um Releases nach Sprache zu filtern.
- [Unpackerr — Documentation](https://unpackerr.zip/docs/introduction)
    und [Docker Image](https://hub.docker.com/r/golift/unpackerr)
    — Entpackt archivierte Downloads für die *Arr*-Apps automatisch.
- [Bazarr (LinuxServer.io)](https://docs.linuxserver.io/images/docker-bazarr/#usage)
    — Begleit-App, die Untertitel für Sonarr- und Radarr-Bibliotheken verwaltet.

## Plex

Ressourcen zu Konfiguration, Transcoding und Fehlerbehebung für Plex.

- [Using Hardware-Accelerated Streaming (Plex Support)](https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/)
    — Wie man Hardware-Transcoding aktiviert und überprüft.
- [Media Capabilities Supported by Intel Hardware](https://www.intel.com/content/www/us/en/docs/onevpl/developer-reference-media-intel-hardware/1-1/overview.html#DECODE-OVERVIEW-11-12)
    — Referenz dazu, welche Codecs deine Intel-iGPU dekodieren/kodieren kann.
- [Plex Codec Issue (TrueNAS Community)](https://www.truenas.com/community/threads/plex-codec-issue.98186/)
    — Fehlerbehebungs-Thread für codec-bedingte Transcoding-Fehler.
- [Can't access Plex from the Android app (r/PleX)](https://www.reddit.com/r/PleX/comments/r0jzrh/cant_access_plex_server_with_android_app_but_web/)
    — Lösungen für das häufige Problem «funktioniert im Browser, aber nicht in der App».

### Alternative

- [Jellyfin](https://jellyfin.org/)
    — Macht (im Grunde) dasselbe wie Plex, ist aber quelloffen (=> kostenlos).

## Usenet

Anleitungen und Dienste für die Download-Seite des Stacks.

- [PCJones Usenet-Guide (German)](https://github.com/PCJones/usenet-guide)
    — Ein ausgezeichneter deutschsprachiger Einsteiger-Guide zum Usenet.
- [NZBGeek](https://www.nzbgeek.info/)
    — Ein beliebter NZB-Indexer.
- [NzbPlanet](https://nzbplanet.net/)
    — Ein weiterer beliebter Indexer mit Lifetime-Plan, gut für englische Titel.
- [treasure-maps (SceneNZBs)](https://treasure-maps.com/)
    — Ein weiterer NZB-Indexer, beliebte Wahl für deutsche Tonspuren.

Übliche Usenet-Anbieter, die während der Einrichtung erwähnt wurden, sind
[TweakNews](https://www.tweaknews.eu/) und
[Fast Usenet](https://www.fastusenet.org/).

## Benachrichtigungen

- [Gotify](https://gotify.net/)
    — Ein selbst gehosteter Server zum Senden und Empfangen von
    Push-Benachrichtigungen, praktisch, um Warnungen deiner Dienste einzurichten.

## Streaming-Technologie (Weiterführende Lektüre)

Hintergrundmaterial, um zu verstehen, wie Media-Streaming unter der Haube
tatsächlich funktioniert.

- [howvideo.works](https://howvideo.works/)
    — Eine kompakte Einführung in digitales Video und Streaming.
- [FFmpeg (Trac)](https://trac.ffmpeg.org/)
    — Dokumentation für das Werkzeug, das nahezu allem Transcoding zugrunde liegt.
- [HTTP Live Streaming (Apple Developer)](https://developer.apple.com/documentation/http-live-streaming)
    — Die HLS-Spezifikation und Entwicklerdokumentation.
- [HTTP Headers (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers)
    und [HTTP Authentication (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication)
    — Referenzen für die HTTP-Schicht, die von Proxys und APIs verwendet wird.
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
    — Das Autorisierungs-Framework hinter vielen Dienstintegrationen.

## Rechtliches

- [Swiss Copyright Act (URG), SR 231.1 — Art. 19 (Fedlex)](https://www.fedlex.admin.ch/eli/cc/1993/1798_1798_1798/de#art_19)
    — Die schweizerische Rechtsgrundlage für Kopien zum Eigengebrauch. Kenne die
    Regeln, die in deiner Rechtsordnung gelten.
