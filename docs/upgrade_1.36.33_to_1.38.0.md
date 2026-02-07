# ZoneMinder Upgrade Guide: 1.36.33 to 1.38.0

## Overview

ZoneMinder 1.38.0 "Seek and Destroy" (released February 1, 2025) is a major release
representing the accumulated work of the entire 1.37.x development branch (~79 database
schema updates). It introduces significant architectural changes, new streaming protocols,
a redesigned permission system, and a fundamental rethinking of how monitor functions work.

This document covers what changed between 1.36.33 and 1.38.0, and what you need to do
to upgrade.

---

## Table of Contents

1. [Interim 1.36.x Releases (1.36.34 - 1.36.37)](#interim-136x-releases-13634---13637)
2. [Major Changes in 1.38.0](#major-changes-in-1380)
   - [Monitor Function Redesign](#1-monitor-function-redesign-breaking-change)
   - [Role-Based Access Control](#2-role-based-access-control-rbac)
   - [Modern Streaming Protocols](#3-modern-streaming-protocols-webrtc-go2rtc-rtsp2web)
   - [Hardware-Accelerated Encoding](#4-hardware-accelerated-encoding)
   - [Camera Integration (ONVIF, MQTT, Amcrest)](#5-enhanced-camera-integration-onvif-mqtt-amcrest)
   - [Event Management & Tagging](#6-event-management--tagging)
   - [Server Monitoring](#7-server-performance-monitoring)
   - [Geolocation](#8-geolocation-support)
   - [UI/UX Changes](#9-uiux-changes)
   - [Performance & Analysis Optimizations](#10-performance--analysis-optimizations)
   - [Storage & Recording](#11-storage--recording)
3. [Breaking Changes Summary](#breaking-changes-summary)
4. [Upgrade Procedure](#upgrade-procedure)
5. [Post-Upgrade Checklist](#post-upgrade-checklist)
6. [Known Issues](#known-issues)
7. [Sources](#sources)

---

## Interim 1.36.x Releases (1.36.34 - 1.36.37)

If you are running exactly 1.36.33, note that several maintenance releases occurred in
the 1.36.x branch before 1.38.0:

### 1.36.34 (August 2023)
- Debian Bookworm support
- FFmpeg 5 channel layout deprecation fixes
- Character set migration to `utf8mb4`
- Improved UI modals
- Multiple security patches (GHSA vulnerabilities)

### 1.36.35 (October 2023)
- FFmpeg 5/6/7 compatibility improvements
- Database connection auto-reconnect
- Filter optimization for database efficiency
- Additional security vulnerability fixes

### 1.36.36 (October 2024)
- Debian Trixie support
- AV1 codec support
- FFmpeg 5.1+ rotation handling
- Enhanced stream scaling
- Security fixes

### 1.36.37 (December 2024)
- FFmpeg 8.0 compatibility
- Migration from libpcre to libpcre2
- Improved zmaudit.pl error handling
- Enhanced zmrecover.pl for deep storage support

---

## Major Changes in 1.38.0

### 1. Monitor Function Redesign (BREAKING CHANGE)

**This is the single most significant change in 1.38.0.** The traditional `Function` field
that controlled all aspects of a monitor's behavior has been split into three independent
settings.

#### Old Model (1.36.x)

A single `Function` enum controlled everything:

| Function | Behavior |
|----------|----------|
| `None` | Monitor disabled |
| `Monitor` | Live view only, no recording or detection |
| `Modect` | Motion detection, record on motion |
| `Record` | Continuous recording, no motion detection |
| `Mocord` | Continuous recording with motion detection |
| `Nodect` | Record on external trigger, no built-in detection |

#### New Model (1.38.0)

Three independent settings replace the single Function field:

| Setting | Values | Purpose |
|---------|--------|---------|
| **Capturing** | `None` / `Ondemand` / `Always` | Whether and when to capture video from the camera |
| **Analysing** | `None` / `Always` | Whether to run motion detection/analysis |
| **Recording** | `None` / `OnMotion` / `Always` | When to save video to disk |

#### Automatic Migration Mapping

The database upgrade automatically maps old values:

| Old Function | Capturing | Analysing | Recording |
|-------------|-----------|-----------|-----------|
| `None` | None | None | None |
| `Monitor` | Always | None | None |
| `Modect` | Always | Always | OnMotion |
| `Record` | Always | None | Always |
| `Mocord` | Always | Always | Always |
| `Nodect` | Always | None | OnMotion |

#### New Possibilities

This redesign enables configurations that were previously impossible:
- **Analyse but don't record** — useful for alerting-only setups
- **On-demand capture** — only pull video when someone is watching or a trigger fires
- **Record without analysis** — for compliance/archival without CPU cost of detection
- **Capture + analyse without recording** — motion alerts with no storage use

#### Impact on Custom Scripts

If you have any custom scripts, API integrations, or automations that reference the
`Function` column in the `Monitors` table, they **will need to be updated** to use the
new `Capturing`, `Analysing`, and `Recording` columns instead.

---

### 2. Role-Based Access Control (RBAC)

The user permission system has been completely rewritten:

- **New database tables**: `User_Roles`, `Role_Groups_Permissions`, `Role_Monitors_Permissions`
- **Reusable role templates** for assigning consistent permissions across users
- **Legacy comma-separated monitor ID strings** replaced with normalized permission tables
- **Fine-grained permissions** for both monitor groups and individual monitors
- **New user profile fields**: Name, Email, Phone
- **User-specific montage layouts** for personalized dashboards
- **Private/system configuration flags** for improved security controls

If you have scripts or external tools that manipulate user permissions directly in the
database, these will need to be updated for the new schema.

---

### 3. Modern Streaming Protocols (WebRTC, Go2RTC, RTSP2Web)

1.38.0 adds support for modern low-latency streaming:

- **Janus WebRTC Gateway** — view live H.264 streams via WebRTC with audio support
- **Go2RTC** — alternative streaming server, widely regarded as the best option for
  H.264 live viewing with audio; requires running go2rtc as a service and configuring
  `GO2RTC_PATH` in Options -> System
- **RTSP2Web** — streaming via WebRTC, MSE (Media Source Extensions), or HLS
- **Configurable codec selection** and stream profiles
- **Default player selection** for preferred streaming client

#### Go2RTC Setup

Go2RTC runs as a separate service. After installing it:
1. Create a `go2rtc.yaml` config with API origin and RTSP listen settings
2. Set `GO2RTC_PATH` in ZoneMinder Options -> System to point to your go2rtc instance

#### Janus Setup

Janus WebRTC Gateway requires separate installation and configuration. Enable per-monitor
in the monitor's Viewing tab under "Janus Live Stream."

---

### 4. Hardware-Accelerated Encoding

- **GPU encoding support** for video, reducing CPU usage significantly
- New configuration options: `EncoderHWAccelName` and `EncoderHWAccelDevice`
- **Human-readable codec names** replacing integer-based codec selection
- Requires compatible GPU and drivers (e.g., NVIDIA NVENC, Intel QSV, AMD AMF/VCE)

---

### 5. Enhanced Camera Integration (ONVIF, MQTT, Amcrest)

- **ONVIF Event Listener** — receive motion/event notifications directly from
  ONVIF-compatible cameras, enabling camera-side motion detection
- **MQTT integration** — publish events to an MQTT broker for consumption by home
  automation platforms (Home Assistant, Node-RED, etc.); requires a running MQTT broker
- **Amcrest API support** — native integration with Amcrest-branded cameras
- **Camera manufacturer and model database** — improved camera management with
  manufacturer/model labeling in monitor configuration

> **Note**: MQTT was reported as causing segfaults on some systems during early 1.38.0
> adoption. If you experience crashes after upgrading, try disabling MQTT on all cameras
> as a diagnostic step.

---

### 6. Event Management & Tagging

- **Event tagging system** — apply custom tags/labels to events for flexible organization
  and filtering (many-to-many relationship between events and tags)
- **Event start/end command execution** — run shell commands when events begin or end,
  reducing dependency on zmeventnotification for some use cases
- **Multiple event close modes** — system, time, duration, idle, alarm-based
- **Section length warnings** for long recordings
- **Filter execution intervals** for scheduled automation
- **Event metadata extensibility** with custom metadata storage

---

### 7. Server Performance Monitoring

- Real-time CPU usage tracking (User, Nice, System, Idle percentages)
- Memory and swap utilization metrics
- Timestamped performance data collection
- Viewable in the ZoneMinder web UI

---

### 8. Geolocation Support

- Geographic coordinates (latitude/longitude) for monitors, events, and servers
- Location-based event tracking and analysis

---

### 9. UI/UX Changes

#### Montage View
- **Expanded grid layouts**: 1, 2, 4, 5, 6, 7, 8, 9, 10, 12, and 16 columns wide
- **User-specific montage layouts** — each user can have their own preferred layout
- Options to locate/hide status information to maximize screen space

#### Watch/Cycle Views
- **Watch and Cycle views have been merged** into a single unified view
- Redesigned interface with improved PTZ controls
- Toggling visibility of PTZ controls
- FPS control in watch view

#### Event Viewing
- Button to jump from event view to montage review at a specific timestamp
- Default player selection preference
- Enhanced video playback controls

#### Filtering
- **Inline filtering** in event list header (no page reload required)
- Log filters at top for component/level filtering

#### File Explorer
- New file explorer view, limited to defined storage areas

#### Decoding Options (Performance)
New decoding modes in the UI for per-monitor tuning:
- **Keyframes Only** — decode only keyframes (I-frames), much lower CPU
- **On Demand** — decode only when someone is viewing
- **Ondemand + Keyframes** — combines both (users report reducing load average from
  17-20 down to 1.5-2.5 with nine cameras)

---

### 10. Performance & Analysis Optimizations

- **Y-Channel analysis** — option to run motion detection on the luminance channel only
  (instead of full color), significantly reducing CPU usage
- **Monitor startup delay** — staggers monitor initialization to reduce load spikes
  on system boot
- **Optimal MJPEG scaling** — reduces CPU/bandwidth while maintaining fidelity
- **Wall clock timestamp synchronization** for passthrough mode
- **Monitor importance levels** (Normal/Less/Not) for stream prioritization

---

### 11. Storage & Recording

- **Email notification format options**: Individual or Summary
- **Improved monitor soft delete** — logical deletion flags rather than physical removal;
  may affect custom scripts that query monitor data
- **Enhanced event notification formatting**

---

## Breaking Changes Summary

| Change | Impact | Action Required |
|--------|--------|----------------|
| Monitor `Function` replaced by `Capturing`/`Analysing`/`Recording` | Automatic DB migration, but scripts/API calls referencing `Function` will break | Update any custom scripts, API integrations, or external tools |
| Permission system normalized | Comma-separated monitor ID strings no longer used | Update any direct DB queries for user permissions |
| Some config parameters renamed/restructured | May affect custom configurations | Review Options after upgrade |
| Monitor soft delete with logical deletion flags | Queries expecting hard-deleted monitors may behave differently | Update custom queries if applicable |
| Database schema (~79 migrations) | Automatic but significant | **Backup database before upgrading** |

---

## Upgrade Procedure

### Prerequisites

- **Supported OS**: Ubuntu 18.04, 20.04, 22.04, or 24.04 (via PPA); Debian Bullseye,
  Bookworm, or Trixie
- **LAMP stack**: Apache, MariaDB/MySQL, PHP
- FFmpeg (1.36.37 already supports up to FFmpeg 8.0)

### Pre-Upgrade Steps

1. **Back up your database** (critical — 79 schema migrations will be applied):
   ```bash
   mysqldump -u root -p --databases zm > zm_backup_$(date +%Y%m%d).sql
   ```

2. **Back up your ZoneMinder configuration**:
   ```bash
   cp -a /etc/zm /etc/zm.bak.$(date +%Y%m%d)
   ```

3. **Stop ZoneMinder**:
   ```bash
   sudo systemctl stop zoneminder
   ```

4. **Document your current monitor Functions** (for reference during post-upgrade
   verification):
   ```bash
   mysql -u root -p zm -e "SELECT Id, Name, Function FROM Monitors;"
   ```

### Upgrade Steps (Ubuntu via PPA)

```bash
# Remove old PPA (substitute your current version)
sudo add-apt-repository --remove ppa:iconnor/zoneminder-1.36

# Add new PPA
sudo add-apt-repository ppa:iconnor/zoneminder-1.38

# Update and upgrade
sudo apt update
sudo apt upgrade -y

# Reload systemd
sudo systemctl daemon-reload

# Start ZoneMinder
sudo systemctl start zoneminder
```

### Upgrade Steps (Debian)

Refer to the [ZoneMinder Wiki](https://wiki.zoneminder.com/) for Debian-specific
instructions. At the time of the 1.38.0 release, the Debian wiki page was still being
updated, and zmrepo packages were not yet available for all distributions.

### Database Migration

The database schema upgrade runs **automatically** when ZoneMinder starts after the
package upgrade. The ~79 schema migrations will be applied in sequence. This may take
several minutes depending on your database size.

You can also trigger it manually:
```bash
sudo zmupdate.pl
```

---

## Post-Upgrade Checklist

- [ ] **Verify monitors are running**: Check that all monitors have started and are
      capturing video. The old Function values should have been automatically mapped to
      the new Capturing/Analysing/Recording settings.
- [ ] **Review monitor settings**: Visit each monitor's settings and confirm the new
      Capturing/Analysing/Recording values match your expectations (see the migration
      mapping table above).
- [ ] **Check user permissions**: If you have multiple users, verify their permissions
      are intact after the RBAC migration.
- [ ] **Test event recording**: Trigger a test event and verify it records correctly.
- [ ] **Review Options**: Go to Options and review any new or renamed configuration
      parameters.
- [ ] **Update custom scripts**: If you have any scripts that query the ZoneMinder
      database or API, update references to the old `Function` field and permission
      structures.
- [ ] **Consider new features**: Evaluate whether to enable go2rtc/Janus for live
      streaming, MQTT for home automation integration, or Y-channel analysis for CPU
      savings.
- [ ] **Check zmeventnotification**: If you use zmeventnotification/zmeventserver,
      verify compatibility and check its
      [CHANGELOG](https://github.com/ZoneMinder/zmeventnotification/blob/master/CHANGELOG.md)
      for any required updates.
- [ ] **Test MQTT carefully**: If enabling MQTT, test with a single camera first — early
      reports indicated MQTT could cause segfaults on some systems.
- [ ] **Explore new decoding options**: Try "Ondemand + Keyframes" decoding mode for
      significant CPU savings on busy systems.

---

## Known Issues

1. **MQTT causing segfaults**: Early adopters on Debian Trixie reported segmentation
   faults (signal 11) related to MQTT. The workaround is to disable MQTT on all cameras
   until the issue is resolved in a point release.

2. **Debian wiki/packages incomplete at launch**: At the 1.38.0 release, Debian
   installation docs and zmrepo packages were still being finalized.

3. **Axis camera MP4 playback**: Some users reported MP4 video from Axis cameras
   stopped working after the 1.38 upgrade.

4. **Montage Review layout issues**: Some layout issues in the Montage Review page were
   reported after upgrade.

---

## Sources

- [ZoneMinder GitHub Releases](https://github.com/ZoneMinder/zoneminder/releases)
- [ZoneMinder 1.38.0 Release Notes (GitHub)](https://github.com/ZoneMinder/zoneminder/releases/tag/1.38.0)
- [Released 1.38 Seek and Destroy - ZoneMinder Forums](https://forums.zoneminder.com/viewtopic.php?t=34262)
- [Upgrade Ubuntu to ZoneMinder 1.38.0 - Forums](https://forums.zoneminder.com/viewtopic.php?t=34261)
- [Upgrade Issues Discussion (Debian Trixie) - Forums](https://forums.zoneminder.com/viewtopic.php?t=34263)
- [Features in 1.37 - ZoneMinder Forums](https://forums.zoneminder.com/viewtopic.php?t=31731)
- [ZoneMinder 1.38.x Ubuntu Wiki](https://wiki.zoneminder.com/Ubuntu_Server_or_Desktop_Zoneminder_1.38.x)
- [ZoneMinder ReadTheDocs - Viewing Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_viewing.html)
- [zmeventnotification GitHub](https://github.com/ZoneMinder/zmeventnotification)
- [ZoneMinder 1.38.0 on newreleases.io](https://newreleases.io/project/github/ZoneMinder/zoneminder/release/1.38.0)
