# Feature: Update to ZoneMinder 1.38.0

You must read, understand, and follow all instructions in `./README.md` when planning and implementing this feature.

## Overview

This project is designed for ZoneMinder ("ZM") 1.36.33 running on Debian 12.2. ZM 1.38.0 was just released (see https://github.com/ZoneMinder/zoneminder/releases/tag/1.38.0 and related ZoneMinder documentation, as well as https://github.com/ZoneMinder/zmdockerfiles for inspiration and tips as well as https://github.com/ZoneMinder/pyzm and https://github.com/ZoneMinder/zmeventnotification). We need to update this Docker image to ZoneMinder 1.38.0 as well as the latest Debian base image and the latest version of all relevant dependencies. There may be major changes from ZM 1.36 to 1.38, so we must analyze all documentation, release notes, etc. for relevance to this project. Our goal is to end up with a Docker image using the latest ZM and dependencies. We will also update `docker-compose.yml` and `docker-compose-mlapi.yml` to use the latest MariaDB images.

Our acceptance criteria are:

1. ZM, the base image, and all dependencies are their latest versions.
2. The Docker image builds successfully.
3. `docker-compose.yml` stands up successfully and serves ZM properly via HTTP.
4. `docker-compose-mlapi.yml` does the same, and all containers start and run successfully.

## Research Summary

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Base image | `debian:13.3` (Trixie) | Debian 13 is now stable; ZM 1.38 has no Bookworm packages |
| ZM installation | Build from source (git tag 1.38.0) | No prebuilt packages exist for any Debian version yet |
| MariaDB | `mariadb:11.8` (LTS) | Current 11.1 is EOL; 11.8 LTS supported until June 2028 |
| ZMES | Keep, fix issues as found | Last release v6.1.29 (Oct 2023); ZM 1.38 compat unverified |
| go2rtc | Include (v1.9.14) | Single binary; provides WebRTC/MSE/HLS streaming for ZM 1.38 |
| PHP version | 8.4 (Trixie default) | Path changes: `/etc/php/8.4/apache2/php.ini` |

### ZM 1.38.0 Major Changes from 1.36

- **Monitor Function redesign**: Single `Function` field split into `Capturing`/`Analysing`/`Recording`
- **RBAC**: Role-based access control replaces legacy permissions
- **New features**: WebRTC (go2rtc), ONVIF events, MQTT, event tagging, server monitoring
- **79 database schema migrations** from 1.36 to 1.38
- **New optional deps**: mosquitto (MQTT), gSOAP (ONVIF), nlohmann_json, libunwind
- **go2rtc** is optional and external (not compiled in); will be included as a bundled binary

### Known Risks

- **MQTT segfault on Trixie**: Forum reports of ZM 1.38 crashing with MQTT enabled on Trixie. Build with MQTT support but note this risk.
  - https://forums.zoneminder.com/viewtopic.php?p=139150
- **ZMES + ZM 1.38**: The monitor Function redesign (single `Function` field split into `Capturing`/`Analysing`/`Recording`) may break ZMES hooks. ZMES last release (v6.1.29, Oct 2023) predates 1.38.0 by over a year. Will test and fix.
  - ZM 1.38 release notes: https://github.com/ZoneMinder/zoneminder/releases/tag/1.38.0
  - ZMES latest release: https://github.com/ZoneMinder/zmeventnotification/releases/tag/v6.1.29
- **Database upgrade**: 79 schema migrations; `zmupdate.pl -f` should handle this but may hit the "Incorrect datetime" bug (workaround: TRUNCATE Monitor_Status).
  - https://forums.zoneminder.com/viewtopic.php?t=34263

## Implementation Plan

### Milestone 1: Multi-stage Dockerfile for ZM 1.38.0 on Trixie

Rewrite the Dockerfile as a multi-stage build: a builder stage that compiles ZM 1.38.0 from source, and a runtime stage with only what's needed to run.

**Task 1.1: Create builder stage**
- Base: `debian:13.3`
- Install build-essential, cmake, git, and all `-dev` packages needed by ZM's CMakeLists.txt
- `git clone --branch 1.38.0 --depth 1 --recurse-submodules` the ZM repo
- Run cmake with Debian-standard paths (`-DCMAKE_INSTALL_PREFIX=/usr`, `-DZM_WEBDIR=/usr/share/zoneminder/www`, `-DZM_CGIDIR=/usr/lib/zoneminder/cgi-bin`, `-DZM_CONFIG_DIR=/etc/zm`, etc.) and Docker-appropriate flags (`-DZM_SYSTEMD=OFF`, `-DBUILD_MAN=OFF`)
- `make -j$(nproc)` and `make DESTDIR=/zminstall install`

**Task 1.2: Create runtime stage**
- Base: `debian:13.3`
- Install runtime packages: Apache 2 + PHP 8.4 modules, ffmpeg, Perl runtime modules, MariaDB client, s6, shared libraries
- `COPY --from=builder /zminstall /` to bring in compiled ZM
- Create ZM user directories, set permissions
- Manually create `/etc/zm/zm.conf` directory structure (no Debian package postinst to do this)

**Task 1.3: Integrate existing content files**
- Copy and install Apache config (`zm-site.conf`), s6 service scripts, `zmcustom.conf`, `status.conf`
- Apache module enablement (rewrite, cgi, headers, expires)
- Verify paths in `zm-site.conf` match cmake install paths; update if needed

**Task 1.4: ZMES integration in Dockerfile**
- Install ZMES Perl dependencies (`Net::WebSocket::Server` via cpanm)
- Install ZMES Python dependencies (pyzm, hook helpers via pip)
- Copy ZMES files to their runtime locations

**Task 1.5: go2rtc integration**
- Download go2rtc v1.9.14 linux/amd64 binary to `/usr/local/bin/go2rtc`
- Create `content/go2rtc-run` s6 service script (exec go2rtc with `-c /etc/zm/go2rtc.yaml`)
- Create `content/go2rtc.yaml` minimal config (API on `:1984`, RTSP on `:8554`, WebRTC on `:8555`)
- Install s6 service and config in Dockerfile
- Expose ports 1984 (API/WebSocket) and 8555 (WebRTC) in Dockerfile
- Note: `ZM_GO2RTC_PATH` must be set by the user in ZM Options → System to the externally-reachable URL (e.g. `http://<docker-host>:1984`), since browsers need to connect to it too

**Task 1.6: Update entrypoint.sh**
- Change PHP ini path from `/etc/php/8.2/apache2/php.ini` to `/etc/php/8.4/apache2/php.ini`
- Verify all other paths still work with source-built ZM (zm.conf location, zm_create.sql, triggers.sql, zmupdate.pl)
- Add note about Monitor_Status truncation workaround for upgrades from 1.36

**Task 1.7: Build and verify**
- `docker build -t docker-zoneminder:dev .` must succeed
- Commit all Milestone 1 changes

### Milestone 2: Docker Compose and Config Updates

**Task 2.1: Update docker-compose.yml**
- MariaDB image: `mariadb:11.1-jammy` → `mariadb:11.8`
- Remove deprecated `version: '2'` key
- Use `depends_on` instead of `links` for zm→mariadb relationship
- Add port mappings for go2rtc: `1984:1984` (API/WebSocket) and `8555:8555` (WebRTC)

**Task 2.2: Update docker-compose-mlapi.yml**
- Same MariaDB and compose format changes as Task 2.1

**Task 2.3: Update example config files if needed**
- Review `*.EXAMPLE.*` files for any ZM 1.38-specific changes (new config keys, changed defaults)

### Milestone 3: CI/CD Updates

**Task 3.1: Update GitHub Actions workflow versions**
- Update `actions/checkout` from v3 to v4
- Update `docker/setup-buildx-action` from v2 to v3
- Update `docker/login-action` from v2 to v3
- Update `docker/build-push-action` from v4 to v6
- Update `actions/create-release` from v1 to current (or switch to a maintained alternative)

**Task 3.2: Verify CI build works**
- Push to feature branch, confirm build succeeds in CI

### Milestone 4: Acceptance Criteria

**Task 4.1: Update documentation**
- Update `README.md` to reflect new versions (Debian 13, ZM 1.38, MariaDB 11.8, PHP 8.4)
- Update `CLAUDE.md` to reflect new architecture (multi-stage build, source compilation, new paths)

**Task 4.2: Final verification and cleanup**
- Verify Docker image builds
- Open/update PR with all changes

**Task 4.3: Move feature document**
- Move `docs/features/update-1.38.0.md` to `docs/features/completed/`

## Progress

- [x] Milestone 1: Multi-stage Dockerfile for ZM 1.38.0 on Trixie
  - Builder stage compiles ZM 1.38.0 from source with cmake on debian:13.3
  - Runtime stage includes Apache + PHP 8.4, all Perl/Python runtime deps, ZMES, go2rtc v1.9.14
  - go2rtc runs as s6 service with config at /etc/zm/go2rtc.yaml
  - entrypoint.sh updated for PHP 8.4 path
  - Docker image builds successfully
- [ ] Milestone 2: Docker Compose and Config Updates
- [ ] Milestone 3: CI/CD Updates
- [ ] Milestone 4: Acceptance Criteria
