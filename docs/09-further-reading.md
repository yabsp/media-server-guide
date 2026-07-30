# Further Links & Related Guides

This chapter collects the external resources, guides and references that
informed this setup. They are a good starting point if you want to dig deeper
into a specific topic or diverge from the choices made in this guide.

## Setup Guides & Platforms

Broader guides covering full media server stacks and alternative platforms.

- [Docker Media Server (SmartHomeBeginner)](https://www.smarthomebeginner.com/docker-media-server-2022/)
    — A comprehensive walkthrough of a Docker-based media server with 20+ apps.
- [Detailed Guide to Setting Up ZFS RAID on Ubuntu](https://manishrjain.com/zfs-raid-ubuntu)
    — An alternative storage approach to the MergerFS pool used in
    [Storage Setup](03-storage.md).
- [CasaOS](https://github.com/IceWhaleTech/CasaOS)
    — A simple, elegant open-source Personal Cloud system if you prefer a
    turnkey UI over a hand-rolled stack.

## Arr Apps & Automation

Reference material for the *Arr* ecosystem and the tools that glue it together.

- [TRaSH Guides — Collection of Custom Formats (Radarr)](https://trash-guides.info/Radarr/Radarr-collection-of-custom-formats/)
    — The de-facto standard for quality profiles and custom formats.
- [TRaSH Guides — How to set up Language Custom Formats](https://trash-guides.info/Sonarr/Tips/How-to-setup-language-custom-formats/)
    — Useful for filtering releases by language.
- [Unpackerr — Documentation](https://unpackerr.zip/docs/introduction)
    and [Docker Image](https://hub.docker.com/r/golift/unpackerr)
    — Automatically extracts archived downloads for the *Arr* apps.
- [Bazarr (LinuxServer.io)](https://docs.linuxserver.io/images/docker-bazarr/#usage)
    — Companion app that manages subtitles for Sonarr and Radarr libraries.

## Plex

Configuration, transcoding and troubleshooting resources for Plex.

- [Using Hardware-Accelerated Streaming (Plex Support)](https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/)
    — How to enable and verify hardware transcoding.
- [Media Capabilities Supported by Intel Hardware](https://www.intel.com/content/www/us/en/docs/onevpl/developer-reference-media-intel-hardware/1-1/overview.html#DECODE-OVERVIEW-11-12)
    — Reference for which codecs your Intel iGPU can decode/encode.
- [Plex Codec Issue (TrueNAS Community)](https://www.truenas.com/community/threads/plex-codec-issue.98186/)
    — Troubleshooting thread for codec-related transcoding failures.
- [Can't access Plex from the Android app (r/PleX)](https://www.reddit.com/r/PleX/comments/r0jzrh/cant_access_plex_server_with_android_app_but_web/)
    — Fixes for the common "works in browser, not in app" problem.

### Alternative

- [Jellyfin](https://jellyfin.org/)
    — Does (basically) the same as Plex does but is open source (=> free).

## Usenet

Guides and services for the download side of the stack.

- [PCJones Usenet-Guide (German)](https://github.com/PCJones/usenet-guide)
    — An excellent German-language beginner's guide to the Usenet.
- [NZBGeek](https://www.nzbgeek.info/)
    — A popular NZB indexer.
- [NzbPlanet](https://nzbplanet.net/)
    — Another popular indexer which offers lifetime plan, good for english titles.
- [treasure-maps (SceneNZBs)](https://treasure-maps.com/)
    — Another NZB indexer, popular choice for german audio.

Common Usenet providers referenced during setup include
[TweakNews](https://www.tweaknews.eu/) and
[Fast Usenet](https://www.fastusenet.org/).

## Notifications

- [Gotify](https://gotify.net/)
    — A self-hosted server for sending and receiving push notifications,
    handy for wiring up alerts from your services.

## Streaming Technology (Further Reading)

Background material for understanding how media streaming actually works under
the hood.

- [howvideo.works](https://howvideo.works/)
    — A concise primer on digital video and streaming.
- [FFmpeg (Trac)](https://trac.ffmpeg.org/)
    — Documentation for the tool underlying nearly all transcoding.
- [HTTP Live Streaming (Apple Developer)](https://developer.apple.com/documentation/http-live-streaming)
    — The HLS specification and developer documentation.
- [HTTP Headers (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers)
    and [HTTP Authentication (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication)
    — References for the HTTP layer used by proxies and APIs.
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
    — The authorization framework behind many service integrations.

## Legal

- [Swiss Copyright Act (URG), SR 231.1 — Art. 19 (Fedlex)](https://www.fedlex.admin.ch/eli/cc/1993/1798_1798_1798/de#art_19)
    — The Swiss legal basis for private-use copying. Know the rules that
    apply in your jurisdiction.
