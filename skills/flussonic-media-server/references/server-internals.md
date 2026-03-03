# Flussonic Media Server v26.02 - Internal Reference Guide

**Overview**: Comprehensive documentation of undocumented internal structures, diagnostic tools, and private APIs discovered through live server exploration.

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [CLI Tools](#2-cli-tools)
3. [Contrib Diagnostic Tools](#3-contrib-diagnostic-tools-32-erlang-escripts)
4. [Private API Endpoints](#4-private-api-endpoints-21-endpoints)
5. [Lua Scripting System](#5-lua-scripting-system)
6. [Recent Changes](#6-recent-changes-v2507---v2602)
7. [Internal Schemas](#7-internal-schemas)
8. [Key Config File Examples](#8-key-config-file-examples)

---

## 1. Directory Structure

Flussonic installs to `/opt/flussonic/` with the following layout:

### Core Directories

```
/opt/flussonic/
├── bin/                    # CLI tools and launchers
├── lib/                    # 43 Erlang application modules
├── priv/                   # Sample media files
├── contrib/                # 32 Erlang escript diagnostic tools
├── wwwroot/flu/            # Web UI files
└── share/tessdata/         # Tesseract OCR data for DVB subtitles
```

### bin/ - CLI Tools
- `erl` - Erlang runtime executable
- `escript` - Erlang script interpreter
- `ffmpeg` - Built-in FFmpeg for transcoding
- `ffprobe` - Built-in FFprobe for media analysis
- `validate_config` - Configuration file validator
- `run` - Main launcher script

### lib/ - Erlang Modules (43 total)
Core modules include:
- cluster-1, config2-1, corelib-1, dash-1, decklink-1, dektec-1
- drm-1, dvb-1, dvr-1, events-1, hls-1, iptv-1, live-1
- mosaic-1.0, motion_detector-1.0, mpegts-3.1, mss-1, ndi-1
- onvif-1, rtmp-3.1, rtsp-3.1, srt-1, st2110-0.1.0
- transcoder-4.2, vod-1, web-1, webrtc-1, and others

### priv/ - Sample Media
- `bunny.mp4` - H.264 video sample
- `beepbop.mp4` - Audio/video sample
- `bunny_subs.ts` - MPEG-TS with subtitles

### contrib/ - Diagnostic Tools (32 escripts)
Located in `/opt/flussonic/contrib/` (detailed in section 3)

### wwwroot/flu/ - Web UI
- `admin3/` - Admin dashboard
- `player/` - Media player
- `dvr3/` - DVR interface
- `swagger/` - API documentation UI
- `webrtc/` - WebRTC interface

### share/tessdata/
Tesseract OCR data for extracting DVB subtitles from MPEG-TS streams.

### System Configuration Paths
- Config: `/etc/flussonic/flussonic.conf`
- Logs: `/var/log/flussonic/`
- Runtime: `/var/run/flussonic/` (PID, SNMP files)

---

## 2. CLI Tools

### run - Main Launcher
The primary command to start Flussonic server:

```bash
/opt/flussonic/bin/run [OPTIONS]
```

**Options**:
- `-n <nodename>` - Erlang node name (default: flussonic)
- `-c <config_path>` - Configuration file path (default: /etc/flussonic/flussonic.conf)
- `-l <log_dir>` - Log directory (default: /var/log/flussonic/)
- `-d` - Daemonize (run in background)
- `-noinput` - No interactive input (used by systemd)

**Example**:
```bash
/opt/flussonic/bin/run -c /etc/flussonic/flussonic.conf -l /var/log/flussonic -noinput -d
```

### validate_config
Pre-validates configuration file before applying:

```bash
/opt/flussonic/bin/validate_config /etc/flussonic/flussonic.conf
```

Checks syntax and configuration schema compliance without starting the server.

### Built-in ffmpeg and ffprobe
Flussonic includes compiled FFmpeg tools at:
- `/opt/flussonic/bin/ffmpeg` - For encoding/transcoding operations
- `/opt/flussonic/bin/ffprobe` - For media stream analysis

### escript - Erlang Script Interpreter
Used to run diagnostic tools from contrib/:

```bash
/opt/flussonic/bin/escript /opt/flussonic/contrib/tool_name.erl [ARGS]
```

---

## 3. Contrib Diagnostic Tools (32 Erlang escripts)

Located in `/opt/flussonic/contrib/`. These are compiled BEAM bytecode archives for internal diagnostics and testing.

### System Control & Configuration
- `control.erl` - Server control operations
- `config.erl` - Configuration inspection and manipulation

### Stream Analysis
- `ingest.erl` - Ingest stream testing
- `rtsp_analyze.erl` - RTSP stream analysis
- `rtsp_read.erl` - RTSP stream reader
- `ts_reader.erl` - MPEG-TS stream reader and analyzer
- `ts_timing.erl` - MPEG-TS timing analysis
- `tsgraph.erl` - MPEG-TS graphing tool
- `pcr_inspector.erl` - PCR (Program Clock Reference) inspection
- `frame_tracer.erl` - Media frame tracing (new in v25.11)

### DVR Operations
- `dvr_control.erl` - DVR management operations
- `dvr_inspect.erl` - DVR archive inspection
- `sparse_lock_dvr.erl` - DVR sparse lock management

### Media Capture & Recording
- `record_input.erl` - Record raw input for debugging
- `multicast_capture.erl` - Capture multicast streams for analysis
- `pcaptool.erl` - Packet capture tool
- `rtmpdump` - RTMP stream capture utility

### Media File Validation
- `mp4_validator.erl` - MP4 file validation
- `m4s_debug.erl` - fMP4 segment debugging
- `export_mp4_header.erl` - MP4 header export

### Streaming Formats
- `dash_reader.erl` - DASH stream reader
- `hls_bench.erl` - HLS performance benchmarking
- `mseld_read.erl` - MSE low-delay reader
- `mbr_udp_analyze.erl` - Multibitrate UDP analysis

### DVB & Hardware
- `dvbscan.erl` - DVB scanner for finding services
- `dvbsub_extract.erl` - DVB subtitle extraction

### Utilities
- `simulator.erl` - Stream simulation tool
- `build_mosaic.erl` - Mosaic builder for multi-stream displays
- `raid_util.erl` - RAID management utility
- `grdnt.erl` - Gradient processing tool
- `transcript.erl` - Transcription tool

### Usage Pattern
All tools use the same invocation pattern:

```bash
/opt/flussonic/bin/escript /opt/flussonic/contrib/tool_name.erl [ARGUMENTS]
```

These tools are crucial for troubleshooting, performance analysis, and stream debugging in production environments.

---

## 4. Private API Endpoints (21 endpoints)

Flussonic maintains two API schemas:
- **Public API**: 74 endpoints (schema-v3-public.json)
- **Private API**: 95 endpoints (schema-v3-private.json)

The 21 **private-only endpoints** not exposed in public documentation:

### System Management (8 endpoints)
- `PUT /activate` - Activate Flussonic license
- `GET /license` - Retrieve license information
- `PUT /license` - Update license
- `DELETE /license` - Remove license
- `GET /license/activations` - List all license activations
- `GET /license/activations/{version}` - Get specific activation
- `PUT /license/activations/{version}` - Update activation
- `DELETE /license/activations/{version}` - Revoke activation
- `GET /license/clients` - List licensed clients
- `GET /license/request` - Get license request for offline activation
- `POST /system/restart` - Restart Flussonic server
- `GET /system/updater` - Check available updates
- `POST /system/updater` - Upgrade Flussonic version
- `GET /system/logs_download` - Download server logs as archive
- `POST /system/upload_logs` - Upload logs to vendor support

### TLS & Certificates (3 endpoints)
- `GET /tls/certificate` - Retrieve TLS certificate
- `DELETE /tls/certificate` - Remove TLS certificate
- `PUT /tls/certificate` - Install TLS certificate
- `POST /tls/letsencrypt` - Issue Let's Encrypt certificate

### SSH Agent (2 endpoints)
- `GET /system/ssh_agent` - Get SSH agent configuration
- `PUT /system/ssh_agent` - Update SSH agent
- `DELETE /system/ssh_agent` - Remove SSH agent

### Cluster (2 endpoints)
- `GET /cluster/balancers` - List load balancers
- `GET /cluster/balancers/{name}` - Get balancer details
- `PUT /cluster/balancers/{name}` - Update balancer
- `DELETE /cluster/balancers/{name}` - Remove balancer

### Stream Operations (2 endpoints)
- `POST /streams/{name}/ads/splice_insert` - Insert SCTE-35 ad splice point
- `POST /streams/{name}/inputs/{index}/select` - Force select specific input source

### Admin & UI (3 endpoints)
- `POST /admin_view_token` - Generate admin authentication token
- `PUT /admin_session_save/{session_id}` - Save telemetry session
- `GET /ui_settings` - Get UI configuration settings

### Monitoring (1 endpoint)
- `GET /otel` - Get OpenTelemetry metrics and statistics

---

## 5. Lua Scripting System

Flussonic supports Lua scripting for custom event handling and authentication. Scripts are configured in `flussonic.conf`.

### securetoken.lua - Token-Based Authentication

Implements token generation with SHA1 hashing, time-based expiry, and IP binding.

**API Functions**:
- `generate_token(user_id, ip, expires_in_seconds)` - Create signed token
- `validate_token(token, ip)` - Verify and parse token
- `http_handler()` - Handle HTTP token requests

**Example Usage**:
```lua
local token = securetoken.generate_token("user123", "192.168.1.100", 3600)
-- Token includes: user_id, IP binding, expiry, SHA1 signature
```

### source_lost.lua - Event Handler

Responds to stream events with external integrations (e.g., Slack webhooks).

**Supported Events**:
- `source_lost` - Triggered when stream source disconnects
- `source_ready` - Source becomes available again
- `stream_stopped` - Stream has stopped

**Example - Slack Integration**:
```lua
timer_ref = streamer.timer(10000, function()
  local msg = {text = "Stream issues: " .. table.concat(issues, ", ")}
  http.post("https://hooks.slack.com/...", {}, json.encode(msg))
end)
```

### Available Lua APIs

**Cryptography**:
- `crypto.sha1(string)` - SHA1 hash function

**HTTP Utilities**:
- `http.post(url, headers, body)` - HTTP POST request
- `http.qs_decode(querystring)` - Parse query string

**JSON**:
- `json.encode(table)` - Serialize table to JSON
- `json.decode(string)` - Parse JSON string

**Logging & Timing**:
- `streamer.log(message)` - Write to server log
- `streamer.now()` - Current UNIX timestamp
- `streamer.timer(milliseconds, callback, arg)` - Schedule timer
- `streamer.cancel_timer(ref)` - Cancel scheduled timer

---

## 6. Recent Changes (v25.07 - v26.02)

### v26.02 (February 2026)
- ST 2110 L24 audio input via NMOS discovery
- SimulCrypt encryption (ECM component)
- `streaming_prefix` option for load balancing configuration
- New counter: `errors_ts_jump_restarts`
- MP4 file validation utility added

### v26.01 (January 2026)
- Full NMOS discovery and device management
- GPU-accelerated thumbnail generation
- M4F thumbnail delivery format (eliminates need for multiple thumbnailers)
- SRT retransmit counter in push telemetry

### v25.12 (December 2025)
- Scrambled packet detection in MPEG-TS streams
- New counters: `errors_ts_pat`, `errors_ts_pmt`
- HTTP error metrics: 404, 403, 500 status codes for stream inputs
- CMAF improved duration accuracy

### v25.11 (November 2025)
- 40+ time series stream parameters collected per minute
- DVR telemetry for recording status monitoring
- Aggregated live streams status metrics
- HDR content identification in HLS/DASH playlists
- WebRTC publish/play telemetry
- DVB CSA scrambling with fixed key support
- LL-HLS with HEVC codec support
- `frame_tracer` diagnostic tool
- OpenTelemetry trace context propagation

### v25.10 (October 2025)
- Manual event creation API
- ONVIF Media 2 API for H265 cameras
- On-demand JPEG preview API
- Authentication token via HTTP headers

### v25.09 (September 2025)
- WebRTC WHIP lost packet retransmission and FEC
- `input_availability` alerts
- ClearKey DRM for DASH
- Flapping stream alerts
- Intel QSV deprecated and **removed** from the transcoder. Only NVIDIA NVENC is supported for GPU acceleration
- Multiplexer ad cue accuracy metrics

### v25.08 (August 2025)
- Fixed WebRTC UDP ports configuration
- SCTE-35 splice improvements with IDR insertion at ad boundaries
- Multiplexer dashboard in Retroview UI

---

## 7. Internal Schemas

Located in `/opt/flussonic/lib/web-1/priv/`:
- `schema-v3-private.json` (1MB, 95 paths, 552 schemas) - Full internal API specification
- `schema-v3-public.json` (890KB, 74 paths, 501 schemas) - Public-facing API specification
- `streaming-private.json` - Streaming endpoints schema
- `chassis-private.json` - Hardware chassis management schema

Located in `/opt/flussonic/lib/manager-24.07.2/priv/`:
- `config_external.json` - External configuration backend schema

These schemas define the complete API surface, request/response validation, and configuration options.

---

## 8. Key Config File Examples

Real-world configurations from live servers (API JSON format, NOT config file syntax — see `references/transcoding.md` for config file syntax):

### Transcoder Configuration
```
transcoder {
  hw = "nvenc"
  deviceid = 0
  gop = 60
  fps = 30
  profiles = [
    {name = "144p", bitrate = "250k"},
    {name = "180p", bitrate = "400k"},
    {name = "288p", bitrate = "800k"},
    {name = "360p", bitrate = "1.3M"},
    {name = "432p", bitrate = "2.4M"},
    {name = "720p", bitrate = "3.7M"}
  ]
}
```

### Input Stream Configuration
```
input {
  scte35 = true
  source_timeout = 15
  buffer_size = "8M"
  cc_check = "repeat"
  backup_inputs = [
    {priority = 1, url = "udp://..."},
    {priority = 2, url = "udp://..."}
  ]
}
```

### HLS/Segment Configuration
```
hls {
  segment_count = 30
  segment_duration = 2000  # milliseconds
}
```

### Advanced Transcoder Parameters
```
transcoder_extra_params = [
  "aresample=async=1",
  "tolerate_corrupted=1"
]
```

### Scale & Deinterlace
```
size_strategy {
  scale = true
  background = "blur"
}

deinterlace {
  enabled = true
  rate = "frame"
}
```

---

## Notes for Operators

1. **Diagnostic Tools**: Most contrib tools require Erlang expertise. Use `--help` flag with escript tools for usage information.

2. **Private APIs**: The 21 private endpoints are internal and not guaranteed stable across versions. Use with caution in production.

3. **Lua Scripts**: Store in `/opt/flussonic/lua/` and reference in configuration. Scripts run in sandboxed environment with limited API access.

4. **Schema Updates**: Check schema files after version upgrades, as new endpoints and configuration options may be added to the private API.

5. **Performance**: The 40+ time series parameters collected in v25.11+ provide detailed telemetry. Monitor via `/otel` endpoint for insights.

6. **License Management**: Private license endpoints allow offline activation and multi-device management through the `/license/` endpoints family.

---

**Last Updated**: February 2026 (Flussonic v26.02)
**Reference Type**: Internal Server Architecture and Diagnostic Tools
