# docker-zoneminder

Modern, best-practices Debian-based Zoneminder container

[![Project Status: WIP – Initial development is in progress, but there has not yet been a stable, usable release suitable for the public.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)

**IMPORTANT:** This is a personal project only. PRs are accepted, but this is not supported and "issues" will likely not be fixed or responded to. This is only for people who understand the details of everything invovled, sorry.

This repo attempts to provide a modern, best-practices Docker image for current ZoneMinder versions, using a current Debian version base. The image provides ZoneMinder 1.38.0 (compiled from source) on Debian 13 (Trixie) with Apache + PHP 8.4, and includes [go2rtc](https://github.com/AlexxIT/go2rtc) for WebRTC/MSE/HLS live streaming. It requires an external MySQL/MariaDB server (the example docker-compose files use MariaDB 11.8 LTS). The image is vehemently NOT auto-updating, as doing so in a Docker image is a mortal sin. If you want to update, then pull a newer tag.

**NOTE:** If you want to use the event server, then you'll need to mount the appropriate configuration files in to the image at ``/etc/zm/es_rules.json``, ``/etc/zm/zmeventnotification.ini``, and ``/etc/zm/secrets.ini``; examples are included in this repo.

In addition, the output of `mod_status` is exposed at `/server-status`.

**go2rtc:** The image includes go2rtc for modern WebRTC streaming. To use it, set `ZM_GO2RTC_PATH` in ZoneMinder Options → System to the externally-reachable URL of the go2rtc API (e.g., `http://<your-docker-host>:1984`). Then enable "Go2RTC Live Stream" on individual monitors. Ports 1984 (API/WebSocket) and 8555 (WebRTC) are exposed.

## Usage

### Important Notes

1. This is really only a very simple **demo / example** to show this image working and show what it can do; this method of running is completely unsuitable for real, long-term usage. To use this for real you'll want to set these Docker containers up so they start automatically (i.e. via systemd units), store data in an appropriate place (currently they store data in the directory they're run from), and are properly monitored and backed up (especially backups of the database).
2. I've only tested the following on Linux. It should probably work on Mac. I'm not sure about Windows, I haven't used it since 2006.
3. This image requires a separate, standalone MySQL/MariaDB database. The example docker-compose file runs one, but it's up to you to back the database up as needed.

### Demo via docker-compose

1. Either clone this git repo on the machine where you want to run it, or download the two `docker-compose` files and all of the `EXAMPLE` files to that machine.
2. Remove the `EXAMPLE.` from the example file names, and edit the content of the files as needed. These are all documented elsewhere, and are all related to the ZM Event Notification server (ZMES) and object detection. If you don't care about ZMES and object detection, then these files can just be left as-is.
3. If the `docker-compose` command isn't already available on your system, [install docker-compose](https://docs.docker.com/compose/install/).
4. In whichever docker-compose file you use (or both), change `ghcr.io/jantman/docker-zoneminder:latest` to the newest [versioned tag](https://github.com/jantman/docker-zoneminder/pkgs/container/docker-zoneminder) of the image.
5. From that same directory, `docker-compose up` should start the database and then zoneminder. If you also want the MLAPI object detection, you can use `docker-compose -f docker-compose-mlapi.yml up`

## Development

1. Cut a branch and make some changes. Ideally build the Docker image locally to ensure it builds. Cut a PR. That will trigger a build, and will push the resulting image to Docker Hub with a tag of the commit SHA.
2. Test that image.
3. When the image is verified to work, merge the PR to `main`.
4. Add a new release version tag for main and push it; that will trigger a full release build and release the new version.
