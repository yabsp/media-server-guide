# Homelab Guide

This site collects the guides for my home server and the network around
it. It consists of two independent parts; each can be read on its own.

| Part | Content | For whom |
| --- | --- | --- |
| [Media Server](#media-server) | Debian server with Plex and the *arr apps | Anyone building the server themselves |
| [Home Network](#home-network) | WiFi with UniFi access points | Anyone operating or re-installing the access points |

## Media Server

Setting up a proper Linux server to host a media library can be quite time-consuming — even when using modern tools. This is why I created this guide. It starts with the [Debian installation](01-debian.md).

### How to Read the Guide?

This part is structured in chapters and sections. The chapters are ordered and the order of the chapters and sections within each chapter is exactly the order in which the steps for the setup should be executed.

### Arr Apps and others

Configuration guides for Sonarr, Radarr, Seerr, NZBGet, Unpackerr and Plex in detail are not provided yet. Since user preferences differ for these tools, priority is rather low for these guides — nonetheless this is planned for future updates. Further reading material is provided in [Further Links & Related Guides](09-further-reading.md).

### Legal Notice

This notice applies to the media server part only. Please keep in mind that downloading copyright protected material is not legal in every region. The media server guide is specifically written to follow the [Federal Act on Copyright and Related Rights](https://www.fedlex.admin.ch/eli/cc/1993/1798_1798_1798/en) of Switzerland.

## Home Network

WiFi in the house runs on two Ubiquiti UniFi access points. This part is not needed for the media server and consists of two pages for different readers:

- [UniFi Access Points](12-unifi-access-points.md) — everything about the devices themselves: what the light means, how to check that everything runs, what to do when something fails, and how to reset a device and set it up again. This is the page for everyday use.
- [UniFi Controller (Docker)](11-unifi-controller.md) — the one-time installation of the controller on the Debian server: Docker Compose, ports, database. Only needed when the controller is set up from scratch.

## Work in Progress

This documentation is by no means final and can always be improved further.
Security has been taken into consideration, but there might still be risks and issues with this setup. You are expected to think for yourself about why we do each step and how to do it properly.
