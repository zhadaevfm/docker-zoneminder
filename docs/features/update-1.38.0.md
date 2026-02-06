# Feature Template

You must read, understand, and follow all instructions in `./README.md` when planning and implementing this feature.

## Overview

This project is designed for ZoneMinder ("ZM") 1.36.33 running on Debian 12.2. ZM 1.38.0 was just released (see https://github.com/ZoneMinder/zoneminder/releases/tag/1.38.0 and related ZoneMinder documentation, as well as https://github.com/ZoneMinder/zmdockerfiles for inspiration and tips as well as https://github.com/ZoneMinder/pyzm and https://github.com/ZoneMinder/zmeventnotification). We need to update this Docker image to ZoneMinder 1.38.0 as well as the latest Debian base image and the latest version of all relevant dependencies. There may be major changes from ZM 1.36 to 1.38, so we must analyze all documentation, release notes, etc. for relevance to this project. Our goal is to end up with a Docker image using the latest ZM and dependencies. We will also update `docker-compose.yml` and `docker-compose-mlapi.yml` to use the latest MariaDB images.

Our acceptance criteria are:

1. ZM, the base image, and all dependencies are their latest versions.
2. The Docker image builds successfully.
3. `docker-compose.yml` stands up successfully and serves ZM properly via HTTP.
4. `docker-compose-mlapi.yml` does the same, and all containers start and run successfully.
