# ZoneMinder Upgrade Guide: 1.36.33 to 1.38.0

## Overview

ZoneMinder 1.38.0 "Seek and Destroy" (released February 1, 2025) is a major release
representing the accumulated work of the entire 1.37.x development branch (~79 database
schema updates). It introduces significant architectural changes, new streaming protocols,
a redesigned permission system, and a fundamental rethinking of how monitor functions work.

This document covers what changed between 1.36.33 and 1.38.0, what you need to do
to upgrade, and a complete reference to every new setting and feature so you can
decide what to enable.

---

## Table of Contents

1. [Interim 1.36.x Releases (1.36.34 - 1.36.37)](#interim-136x-releases-13634---13637)
2. [Major Changes in 1.38.0](#major-changes-in-1380)
   - [Monitor Function Redesign (Breaking)](#1-monitor-function-redesign-breaking-change)
   - [Decoding Modes (Major CPU Savings)](#2-decoding-modes-major-cpu-savings)
   - [Y-Channel Analysis](#3-y-channel-analysis)
   - [Modern Streaming: Go2RTC, Janus, RTSP2Web](#4-modern-streaming-go2rtc-janus-rtsp2web)
   - [Hardware Acceleration](#5-hardware-acceleration-decoding--encoding)
   - [ONVIF Events](#6-onvif-event-listener)
   - [MQTT Integration](#7-mqtt-integration)
   - [Amcrest API](#8-amcrest-api-support)
   - [Event Tagging](#9-event-tagging)
   - [Event Start/End Commands](#10-event-startend-commands)
   - [Role-Based Access Control](#11-role-based-access-control-rbac)
   - [Server Performance Monitoring](#12-server-performance-monitoring)
   - [Geolocation](#13-geolocation-support)
   - [UI/UX Changes](#14-uiux-changes)
   - [Recording & Storage](#15-recording--storage-changes)
   - [Filter Enhancements](#16-filter-enhancements)
3. [Complete Per-Monitor Settings Reference](#complete-per-monitor-settings-reference)
4. [System-Level Options Reference](#system-level-options-reference)
5. [New Database Tables](#new-database-tables)
6. [Breaking Changes Summary](#breaking-changes-summary)
7. [Upgrade Procedure](#upgrade-procedure)
8. [Post-Upgrade Checklist](#post-upgrade-checklist)
9. [Known Issues](#known-issues)
10. [Sources](#sources)

---

## Interim 1.36.x Releases (1.36.34 - 1.36.37)

If you are running exactly 1.36.33, several maintenance releases shipped in the 1.36.x
branch before 1.38.0:

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
has been split into three independent settings.

#### Old Model (1.36.x)

| Function | Behavior |
|----------|----------|
| `None` | Monitor disabled |
| `Monitor` | Live view only, no recording or detection |
| `Modect` | Motion detection, record on motion |
| `Record` | Continuous recording, no motion detection |
| `Mocord` | Continuous recording with motion detection |
| `Nodect` | Record on external trigger, no built-in detection |

#### New Model (1.38.0)

| Setting | Values | Where | Purpose |
|---------|--------|-------|---------|
| **Capturing** | `None` / `Ondemand` / `Always` | Source Tab | Whether and when to capture video |
| **Analysing** | `None` / `Always` | Analysis Tab | Whether to run motion detection |
| **Recording** | `None` / `OnMotion` / `Always` | Recording Tab | When to save video to disk |

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

- **Analyse but don't record** (Capturing=Always, Analysing=Always, Recording=None) — alerting-only
- **On-demand capture** (Capturing=Ondemand) — only pull video when someone views or a trigger fires
- **Record without analysis** (Capturing=Always, Analysing=None, Recording=Always) — archival without CPU cost
- **Capture + analyse without recording** — motion alerts with no storage

#### Impact on Custom Scripts

Any custom scripts, API integrations, or automations that reference the `Function` column
in the `Monitors` table **must be updated** to use `Capturing`, `Analysing`, and
`Recording` instead.

---

### 2. Decoding Modes (Major CPU Savings)

**Location: Monitor -> Source Tab -> Decoding**

This is one of the most impactful performance features. The old boolean `DecodingEnabled`
field is replaced by a multi-value enum:

| Value | Description |
|-------|-------------|
| `None` | No frames decoded; live view and thumbnails unavailable |
| `Ondemand` | Decode only when someone is actively viewing the stream |
| `KeyFrames` | Only decode keyframes (I-frames); very low CPU but choppy preview depending on camera keyframe interval |
| `KeyFrames+Ondemand` | Keyframes normally, full decode when a viewer connects |
| `Always` | Decode every frame (default, highest CPU; same as 1.36 behavior) |

**Recommendation**: Try `KeyFrames+Ondemand` on all monitors. Users report reducing
system load average from 17-20 down to 1.5-2.5 with nine cameras.

---

### 3. Y-Channel Analysis

**Location: Monitor -> Analysis Tab -> Analysis Image**

| Value | Description |
|-------|-------------|
| `FullColour` | Motion detection using full RGB image (default, same as 1.36) |
| `YChannel` | Use only the Y (luminance) channel from a YUV-format image — a greyscale analysis that significantly reduces CPU usage |

**Recommendation**: Enable `YChannel` unless you rely on color-based motion detection
zones or filtering.

---

### 4. Modern Streaming: Go2RTC, Janus, RTSP2Web

These are the biggest quality-of-life improvements for live viewing. Instead of MJPEG
streams of JPEG images, you can now view native H.264/H.265 video with audio in
the browser.

#### Go2RTC (Recommended)

The simplest and most widely recommended option for H.264 live viewing with audio.

**Setup**:
1. Install go2rtc binary and run it as a systemd service
2. Create `/opt/go2rtc/go2rtc.yaml` (or wherever you install it):
   ```yaml
   api:
     origin: "*"
     listen: "0.0.0.0:1984"
   rtsp:
     listen: ""
   ```
3. Set **Options -> System -> `GO2RTC_PATH`** to `http://<server_ip>:1984/api`
   - Use an IP/hostname reachable by both the ZM server AND browser clients
   - Do NOT use `127.0.0.1` unless only accessing ZM from the same machine
4. Per-monitor: **Monitor -> Viewing Tab -> Go2RTC Live Stream** = checked

ZoneMinder automatically registers each monitor's stream with go2rtc. Verify at
`http://<server_ip>:1984/`.

#### Janus WebRTC Gateway

Alternative to go2rtc; requires more setup but enables WebRTC with the Janus ecosystem.

**Per-monitor settings (Viewing Tab)**:

| Setting | Type | Description |
|---------|------|-------------|
| **Janus Live Stream** (`JanusEnabled`) | Boolean | Master enable |
| **JanusAudioEnabled** | Boolean | Include audio in the WebRTC stream |
| **Janus_Profile_Override** | String | Override the H.264 encoding profile |
| **Janus_Use_RTSP_Restream** | Boolean | Use ZM's RTSP restream as source instead of camera directly |
| **Janus_RTSP_User** | INT | User ID for RTSP authentication |
| **Janus_RTSP_Session_Timeout** | INT | Session timeout in seconds (0 = default) |

**Setup**:
1. Install Janus WebRTC Gateway
2. Configure `/etc/janus/janus.cfg` and streaming plugin
3. Symlink JS library: `/usr/share/javascript/janus-gateway` -> `/usr/share/javascript/janus`
4. Enable per-monitor via the Viewing tab settings above

#### RTSP2Web

Experimental streaming server supporting three protocols:

**Per-monitor settings (Viewing Tab)**:

| Setting | Type | Description |
|---------|------|-------------|
| **RTSP2Web Live Stream** (`RTSP2WebEnabled`) | Boolean | Enable RTSP2Web |
| **RTSP2Web Type** (`RTSP2WebType`) | Enum | `HLS`, `MSE`, or `WebRTC` |

#### Built-in RTSP Server

ZoneMinder now includes a built-in RTSP server for re-streaming monitor feeds:

| Setting | Where | Description |
|---------|-------|-------------|
| **RTSP Server** | Viewing Tab | Boolean — expose this monitor via ZM's RTSP server |
| **RTSPStreamName** | Viewing Tab | Unique path for the RTSP URL |

#### Other Viewing Tab Settings

| Setting | Description |
|---------|-------------|
| **StreamChannel** | `Restream`, `CameraDirectPrimary`, `CameraDirectSecondary` — source for WebRTC viewers |
| **DefaultPlayer** | Preferred player (e.g., `rtsp2web_webrtc`, `rtsp2web_mse`, `rtsp2web_hls`) |
| **Default Method For Event View** | `MP4` (for H.264) or `MJPEG` (for H.265 requiring conversion) |

---

### 5. Hardware Acceleration (Decoding & Encoding)

#### Decoding Hardware Acceleration (Source Tab)

| Setting | Description | Example Values |
|---------|-------------|----------------|
| **DecoderHWAccelName** | GPU acceleration method | `vaapi` (Intel/AMD), `cuda` (NVIDIA), `vdpau`, `cuvid` |
| **DecoderHWAccelDevice** | GPU device path | `/dev/dri/renderD128` (Intel/AMD); blank for NVIDIA |
| **Decoder** | Specific FFmpeg decoder | Leave blank for auto |

#### Encoding Hardware Acceleration (Recording Tab)

| Setting | Description | Example Values |
|---------|-------------|----------------|
| **EncoderHWAccelName** | GPU acceleration method | `vaapi`, `cuda` |
| **EncoderHWAccelDevice** | GPU device path | `/dev/dri/renderD128` |

#### Output Codec (Recording Tab)

| Setting | Description |
|---------|-------------|
| **OutputCodecName** | Human-readable: `auto`, `h264`, `hevc`, `vp9`, `av1` (replaces old integer OutputCodec) |
| **Output Container** | `auto`, `mp4`, `mkv`, `webm` |
| **Encoder** | Specific encoder including HW-accelerated options |

**Tips**:
- Try leaving device fields blank first; only specify if auto-detection fails
- Requires FFmpeg compiled with the appropriate HW acceleration support
- NVIDIA: ensure CUDA/NVDEC/NVENC drivers are installed
- Intel/AMD: ensure VA-API drivers and `/dev/dri/renderD128` are present

---

### 6. ONVIF Event Listener

**Location: Monitor -> ONVIF Tab (NEW TAB)**

Receive motion/event notifications directly from ONVIF-compatible cameras, enabling
camera-side motion detection instead of (or in addition to) ZoneMinder's CPU-based
analysis.

| Setting | Type | Description |
|---------|------|-------------|
| **ONVIF_URL** | String | `http://user:pass@camera_ip:port/onvif/device_service` |
| **ONVIF_Events_Path** | String (default `/Events`) | Path for pull point subscription; may need adjustment for some cameras |
| **Username** | String | ONVIF authentication username |
| **Password** | String | ONVIF authentication password |
| **ONVIF_Options** | String | Additional ONVIF protocol options |
| **ONVIF_Event_Listener** | Boolean | Enable/disable event listening |
| **ONVIF_Alarm_Text** | String (default `MotionAlarm`) | Alarm text string to match from camera events |
| **SOAP_wsa_compl** | Boolean (default TRUE) | SOAP WS-Addressing compliance; toggle if camera has issues |

Compatible cameras include Hikvision, Amcrest, Reolink, and Tapo. Some Hikvision cameras
require a dedicated ONVIF user with specific permissions.

**Use case**: If your cameras already have good motion detection, you can set
Analysing=None and rely on ONVIF events to trigger recording, saving significant CPU.

---

### 7. MQTT Integration

**Location: Monitor -> MQTT Tab (NEW TAB)**

| Setting | Type | Description |
|---------|------|-------------|
| **MQTT_Enabled** | Boolean | Enable/disable MQTT for this monitor |
| **MQTT_Subscriptions** | String | MQTT subscription topic details |

**Note**: The built-in per-monitor MQTT feature is documented as "not yet fully
implemented" and is under active development. The more mature path for MQTT integration
is through zmeventnotification.pl with the `MQTT::Simple` Perl module and a Mosquitto
broker.

**Warning**: Early adopters on Debian Trixie reported segfaults related to MQTT. Test
with a single camera first.

---

### 8. Amcrest API Support

**Location: Monitor -> Misc Tab**

| Setting | Type | Description |
|---------|------|-------------|
| **use_Amcrest_API** | Boolean | Enable Amcrest-specific API integration for this monitor |

Provides native integration with Amcrest-branded cameras.

---

### 9. Event Tagging

A many-to-many tagging system for organizing events.

**How to use**:
- Tags can be assigned to events through the web UI event view
- Tags are filterable in the Events filter system (new filter condition: `Tags`)
- Multiple tags per event, single tag across many events
- Tags are user-attributed (who created, who assigned, when)

**Database tables**: `Tags` (id, name, creator, dates) and `Events_Tags` (junction table
with cascade delete).

---

### 10. Event Start/End Commands

**Location: Monitor -> Recording Tab**

| Setting | Type | Description |
|---------|------|-------------|
| **Event Start Command** | String (varchar 255) | Shell command executed when an event starts |
| **Event End Command** | String (varchar 255) | Shell command executed when an event ends |

This reduces dependency on zmeventnotification for simple command-execution use cases
(e.g., turning on a light, sending a webhook).

---

### 11. Role-Based Access Control (RBAC)

The user permission system has been completely rewritten:

- **New permission tables**: `Groups_Permissions` and `Monitors_Permissions` replace
  old comma-separated `MonitorIds` strings
- **Per-group and per-monitor permissions**: `Inherit` / `None` / `View` / `Edit`
- **Users.Monitors** column changed to enum: `None` / `View` / `Edit` / `Create`
- **New user profile fields**: `Name`, `Email`, `Phone`
- **User-specific montage layouts** via `UserId` on `MontageLayouts` table
- **Config table flags**: `Private` and `System` booleans for security controls

If you have scripts or external tools that manipulate user permissions directly in the
database, these will need to be updated for the new schema.

---

### 12. Server Performance Monitoring

Automatic real-time tracking of server health (no configuration needed):

| Metric | Column(s) |
|--------|-----------|
| CPU User % | `CpuUserPercent` |
| CPU Nice % | `CpuNicePercent` |
| CPU System % | `CpuSystemPercent` |
| CPU Idle % | `CpuIdlePercent` |
| CPU Usage % | `CpuUsagePercent` |

Stored in both `Server_Stats` and `Servers` tables. Viewable in the web UI.

Related: **Options -> System -> `STATUS_UPDATE_INTERVAL`** controls how often stats
are collected.

---

### 13. Geolocation Support

Geographic coordinates on monitors, events, and servers. The `Longitude` columns use
`DECIMAL(11,8)` for high precision.

---

### 14. UI/UX Changes

#### Montage View
- **Expanded grid layouts**: 1, 2, 3, 4, 6, 8, 12, 16, 24 columns wide
- **Per-user montage layouts** — each user can have their own preferred layout
- Options to locate/hide status information to maximize screen space

#### Watch/Cycle Views
- **Watch and Cycle views merged** into a single unified view
- Redesigned interface with improved/toggleable PTZ controls
- FPS control in watch view

#### Event Viewing
- Button to jump from event view to montage review at a specific timestamp
- Default player selection preference
- Enhanced video playback controls

#### Filtering
- **Inline filtering** in event list header (no page reload)
- Log filters at top for component/level filtering

#### File Explorer
- New file explorer view, limited to defined storage areas

#### General Tab (Monitor)
- **Manufacturer** and **Model** dropdowns for camera metadata
- **Importance**: `Normal` / `Less Important` / `Not Important` — controls stream
  priority and default visibility

---

### 15. Recording & Storage Changes

#### New Recording Tab Fields

| Setting | Type | Description |
|---------|------|-------------|
| **RecordingSource** | Enum | `Primary`, `Secondary`, `Both` — which stream(s) to record |
| **Recording Audio** | Boolean | Enable/disable audio recording |
| **WallClockTimestamps** | Boolean | Use wall clock timestamps during passthrough recording |
| **EventCloseMode** | Enum | `system`, `time`, `duration`, `idle`, `alarm` — how events close in continuous recording |
| **SectionLengthWarn** | Boolean | Warn when section length is excessive |

#### Secondary Stream Support

| Setting | Where | Description |
|---------|-------|-------------|
| **SourceSecondPath** | Source Tab | Secondary stream URL (e.g., lower-res for analysis) |
| **AnalysisSource** | Analysis Tab | `Primary` or `Secondary` — which stream to analyze |
| **RecordingSource** | Recording Tab | `Primary`, `Secondary`, or `Both` |

This enables a powerful pattern: analyze the low-res substream (saving CPU) but record
the high-res main stream.

---

### 16. Filter Enhancements

#### New Filter Table Columns

| Column | Type | Description |
|--------|------|-------------|
| **ExecuteInterval** | INT (default 60) | How often (in seconds) a background filter runs |
| **EmailFormat** | Enum | `Individual` or `Summary` digest |
| **EmailServer** | TEXT | Per-filter email server override |

#### New Filter Condition
- **Tags** — filter events by tag (works with the new tagging system)

#### Related Options
- **Options -> System -> `FILTER_RELOAD_DELAY`** — how often filters are reloaded from DB
- **Options -> System -> `FILTER_EXECUTE_INTERVAL`** — how often filters run against events

---

## Complete Per-Monitor Settings Reference

This is a consolidated reference of **all new fields** added to monitor configuration
tabs in 1.37/1.38, so you can walk through each tab after upgrading.

### General Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| Manufacturer | Dropdown | Camera manufacturer (informational) |
| Model | Dropdown | Camera model (informational) |
| Importance | `Normal` / `Less` / `Not` | Stream display priority |

### Source Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| **Capturing** | `None` / `Ondemand` / `Always` | Video capture control |
| **Decoding** | `None` / `Ondemand` / `KeyFrames` / `KeyFrames+Ondemand` / `Always` | Frame decode strategy |
| SourceSecondPath | URL string | Secondary stream URL |
| Decoder | String | Specific FFmpeg decoder (blank = auto) |
| DecoderHWAccelName | `vaapi` / `cuda` / `vdpau` / `cuvid` | HW decode method |
| DecoderHWAccelDevice | Path | GPU device path |

### Analysis Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| **Analysing** | `None` / `Always` | Motion detection control |
| **Analysis Image** | `FullColour` / `YChannel` | Color vs luminance-only analysis |
| AnalysisSource | `Primary` / `Secondary` | Which stream to analyze |

### Recording Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| **Recording** | `None` / `OnMotion` / `Always` | When to save video |
| RecordingSource | `Primary` / `Secondary` / `Both` | Which stream(s) to record |
| OutputCodecName | `auto` / `h264` / `hevc` / `vp9` / `av1` | Output codec |
| Encoder | Selection | Encoder (incl. HW options) |
| EncoderHWAccelName | `vaapi` / `cuda` | HW encode method |
| EncoderHWAccelDevice | Path | GPU device path |
| Output Container | `auto` / `mp4` / `mkv` / `webm` | Container format |
| Recording Audio | Boolean | Record audio |
| Event Start Command | String | Shell cmd on event start |
| Event End Command | String | Shell cmd on event end |
| WallClockTimestamps | Boolean | Wall clock in passthrough |
| EventCloseMode | `system` / `time` / `duration` / `idle` / `alarm` | How events close |
| SectionLengthWarn | Boolean | Warn on long sections |

### Viewing Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| RTSP Server | Boolean | Expose via built-in RTSP server |
| RTSPStreamName | String (unique) | RTSP URL path |
| **Janus Live Stream** | Boolean | Enable Janus WebRTC |
| JanusAudioEnabled | Boolean | Audio in Janus stream |
| Janus_Profile_Override | String | H.264 profile override |
| Janus_Use_RTSP_Restream | Boolean | Janus source selection |
| Janus_RTSP_User | INT | RTSP auth user ID |
| Janus_RTSP_Session_Timeout | INT | Session timeout (seconds) |
| **RTSP2Web Live Stream** | Boolean | Enable RTSP2Web |
| RTSP2Web Type | `HLS` / `MSE` / `WebRTC` | RTSP2Web protocol |
| **Go2RTC Live Stream** | Boolean | Enable go2rtc |
| StreamChannel | `Restream` / `CameraDirectPrimary` / `CameraDirectSecondary` | Stream source for WebRTC |
| DefaultPlayer | String | Preferred streaming player |
| Default Method For Event View | `MP4` / `MJPEG` | Event playback format |
| DefaultScale | String | Default viewing scale |

### ONVIF Tab (NEW)
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| ONVIF_URL | String | ONVIF device service URL |
| ONVIF_Events_Path | String (default `/Events`) | Event subscription path |
| Username | String | ONVIF username |
| Password | String | ONVIF password |
| ONVIF_Options | String | Additional options |
| **ONVIF_Event_Listener** | Boolean | Enable event listening |
| ONVIF_Alarm_Text | String (default `MotionAlarm`) | Alarm text to match |
| SOAP_wsa_compl | Boolean (default TRUE) | WS-Addressing compliance |

### MQTT Tab (NEW)
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| MQTT_Enabled | Boolean | Enable MQTT |
| MQTT_Subscriptions | String | Subscription topic details |

### Misc Tab — New Fields
| Field | Values / Type | Purpose |
|-------|---------------|---------|
| StartupDelay | INT (seconds, default 0) | Stagger monitor startup |
| use_Amcrest_API | Boolean | Amcrest camera integration |

---

## System-Level Options Reference

Key new or changed settings under **Options** (gear icon):

### Options -> System — New Settings
| Setting | Description |
|---------|-------------|
| **`GO2RTC_PATH`** | URL to go2rtc API (e.g., `http://192.168.1.10:1984/api`) |
| `STATUS_UPDATE_INTERVAL` | How often server stats are collected |
| `FILTER_RELOAD_DELAY` | How often filters reload from DB |
| `FILTER_EXECUTE_INTERVAL` | How often filters run against events |
| `CASE_INSENSITIVE_USERNAMES` | Case-insensitive login matching |

### Config Table — New Flags
| Flag | Purpose |
|------|---------|
| `Private` | Marks config entries as private/restricted |
| `System` | Marks config entries as system-level |

---

## New Database Tables

| Table | Purpose |
|-------|---------|
| `Tags` | Event tag definitions (name, creator, dates) |
| `Events_Tags` | Many-to-many event-to-tag junction |
| `Groups_Permissions` | Per-group per-user RBAC permissions |
| `Monitors_Permissions` | Per-monitor per-user RBAC permissions |
| `Reports` | Report definitions with filter/date/interval |
| `User_Preferences` | Per-user preference key/value storage |
| `Event_Data` | Extensible metadata storage for events |
| `Object_Types` | Object type definitions (for future detection use) |

---

## Breaking Changes Summary

| Change | Impact | Action Required |
|--------|--------|----------------|
| Monitor `Function` -> `Capturing`/`Analysing`/`Recording` | Auto-migrated, but scripts referencing `Function` break | Update custom scripts and API calls |
| Permission system normalized | `MonitorIds` comma-separated strings removed | Update direct DB queries for permissions |
| `DecodingEnabled` boolean -> `Decoding` enum | Auto-migrated | No action unless scripts reference it |
| `OutputCodec` integer -> `OutputCodecName` string | Auto-migrated | Update scripts referencing codec IDs |
| cURL monitor type removed | Auto-migrated to FFmpeg | No action |
| `Events.Cause` changed from varchar(32) to TEXT | Wider column | No action normally |
| `Snapshot_Events` renamed to `Snapshots_Events` | Table name change | Update scripts referencing old name |
| Config `Private`/`System` flags added | New security controls | Review after upgrade |
| Database schema (~79 migrations) | Automatic but significant | **Backup database before upgrading** |

---

## Upgrade Procedure

### Prerequisites

- **Supported OS**: Ubuntu 18.04/20.04/22.04/24.04 (PPA); Debian Bullseye/Bookworm/Trixie
- **LAMP stack**: Apache, MariaDB/MySQL, PHP
- FFmpeg (1.36.37 already supports up to FFmpeg 8.0)

### Pre-Upgrade Steps

1. **Back up your database** (critical):
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

4. **Document current monitor Functions** (for post-upgrade verification):
   ```bash
   mysql -u root -p zm -e "SELECT Id, Name, Function FROM Monitors;"
   ```

### Upgrade (Ubuntu via PPA)

```bash
sudo add-apt-repository --remove ppa:iconnor/zoneminder-1.36
sudo add-apt-repository ppa:iconnor/zoneminder-1.38
sudo apt update
sudo apt upgrade -y
sudo systemctl daemon-reload
sudo systemctl start zoneminder
```

### Upgrade (Debian)

Refer to the [ZoneMinder Wiki](https://wiki.zoneminder.com/) for Debian-specific
instructions.

### Database Migration

Runs **automatically** on ZoneMinder start (~79 migrations, may take several minutes).
Manual trigger: `sudo zmupdate.pl`

---

## Post-Upgrade Checklist

### Immediate Verification
- [ ] All monitors are running and capturing video
- [ ] Capturing/Analysing/Recording values match expectations (see migration table)
- [ ] User permissions intact after RBAC migration
- [ ] Events are recording correctly
- [ ] Review Options page for new/renamed settings

### Custom Integration Updates
- [ ] Update scripts referencing `Function` column -> use `Capturing`/`Analysing`/`Recording`
- [ ] Update scripts referencing `MonitorIds` -> use `Monitors_Permissions` table
- [ ] Check zmeventnotification compatibility ([CHANGELOG](https://github.com/ZoneMinder/zmeventnotification/blob/master/CHANGELOG.md))

### Performance Optimizations to Try
- [ ] **Decoding mode**: Set to `KeyFrames+Ondemand` on all monitors (Source Tab)
- [ ] **Y-Channel analysis**: Enable `YChannel` on Analysis Tab
- [ ] **Monitor startup delay**: Stagger startup on Misc Tab if boot causes load spikes
- [ ] **Secondary stream**: Configure `SourceSecondPath` for analysis on low-res + record on high-res

### New Features to Evaluate
- [ ] **Go2RTC**: Install go2rtc service, set `GO2RTC_PATH`, enable per-monitor for H.264 live view with audio
- [ ] **Janus WebRTC**: Alternative to go2rtc if you prefer the Janus ecosystem
- [ ] **ONVIF Events**: Enable `ONVIF_Event_Listener` on compatible cameras to offload motion detection to camera
- [ ] **Event Start/End Commands**: Set up shell commands on Recording Tab for automation
- [ ] **Event Tagging**: Organize events with custom tags
- [ ] **Hardware Acceleration**: Set `DecoderHWAccelName`/`EncoderHWAccelName` if you have a GPU
- [ ] **MQTT**: Test cautiously with one camera first if you want broker integration
- [ ] **Amcrest API**: Enable `use_Amcrest_API` for Amcrest cameras
- [ ] **Built-in RTSP Server**: Enable per-monitor to restream via RTSP
- [ ] **Montage layouts**: Try expanded grid options and per-user layouts

---

## Known Issues

1. **MQTT causing segfaults**: Reported on Debian Trixie (signal 11). Workaround: disable
   MQTT on all cameras.

2. **Debian packages incomplete at launch**: Debian wiki/zmrepo packages were still being
   finalized at 1.38.0 release.

3. **Axis camera MP4 playback**: Some users reported MP4 from Axis cameras broke after
   upgrade.

4. **Montage Review layout issues**: Some layout problems reported after upgrade.

---

## Sources

- [ZoneMinder GitHub Releases](https://github.com/ZoneMinder/zoneminder/releases)
- [ZoneMinder 1.38.0 Release Notes](https://github.com/ZoneMinder/zoneminder/releases/tag/1.38.0)
- [Released 1.38 Seek and Destroy - Forums](https://forums.zoneminder.com/viewtopic.php?t=34262)
- [Upgrade Ubuntu to 1.38.0 - Forums](https://forums.zoneminder.com/viewtopic.php?t=34261)
- [Upgrade Issues (Debian Trixie) - Forums](https://forums.zoneminder.com/viewtopic.php?t=34263)
- [ZoneMinder ReadTheDocs](https://zoneminder.readthedocs.io/en/latest/)
  - [Source Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_source.html)
  - [Viewing Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_viewing.html)
  - [Recording Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_recording.html)
  - [Analysis Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_analysis.html)
  - [ONVIF Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_onvif.html)
  - [MQTT Tab](https://zoneminder.readthedocs.io/en/latest/userguide/definemonitor/definemonitor_mqtt.html)
  - [Filter Events](https://zoneminder.readthedocs.io/en/latest/userguide/filterevents.html)
  - [Options](https://zoneminder.readthedocs.io/en/latest/userguide/options.html)
- [ZoneMinder Wiki](https://wiki.zoneminder.com/)
- [go2rtc Forum Thread](https://forums.zoneminder.com/viewtopic.php?t=34044)
- [Janus Forum Thread](https://forums.zoneminder.com/viewtopic.php?t=33320)
- [ZoneMinder DB Migrations (GitHub)](https://github.com/ZoneMinder/zoneminder/tree/master/db/)
- [zmeventnotification GitHub](https://github.com/ZoneMinder/zmeventnotification)
- [ZoneMinder 1.38.0 on newreleases.io](https://newreleases.io/project/github/ZoneMinder/zoneminder/release/1.38.0)
