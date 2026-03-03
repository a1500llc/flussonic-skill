# Flussonic General Concepts Guide

## Table of Contents
- [Core Architecture](#core-architecture)
- [Streams Overview](#streams-overview)
- [Configuration Files](#configuration-files)
- [API Basics](#api-basics)
- [Logging & Monitoring](#logging--monitoring)
- [Performance Concepts](#performance-concepts)
- [Common Workflows](#common-workflows)

## Core Architecture

### Component Overview
```
Flussonic Media Server
├── Input Engine (Ingest)
│   ├── RTSP/RTP listeners
│   ├── RTMP server
│   ├── SRT server
│   ├── HTTP/UDP receivers
│   └── File readers
│
├── Processing Pipeline
│   ├── Transcoding (CPU/GPU)
│   ├── DVR recording
│   ├── Stream muxing
│   └── Filtering
│
└── Output Engine (Playback/Push)
    ├── HLS packager
    ├── DASH packager
    ├── RTMP/RTMPS server
    ├── HTTP server
    ├── WebRTC server
    └── Push/restream clients
```

### Stream Lifecycle
```
Source → Ingest → Process → Deliver
  |        |          |         |
Camera   RTSP    Transcode    HLS/DASH
RTMP     HTTP    DVR Record    RTMP
File     UDP     Mosaic       Multicast
         SRT     Logo/Text     WebRTC
```

## Streams Overview

### What is a Stream?
A stream is a named entity that:
- Receives video/audio from source(s)
- Optionally processes (transcode, record)
- Outputs to one or more destinations
- Maintains state and statistics

### Stream State Diagram
```
         ┌─────────────────┐
         │    Disabled     │
         └────────┬────────┘
                  │
                  │ Stream configured
                  ▼
         ┌─────────────────┐
         │   Stopped/      │
         │   Initializing  │
         └────────┬────────┘
                  │
        Source connection attempt
                  │
          ┌───────┴──────────┐
          │                  │
          ▼                  ▼
    ┌──────────┐    ┌─────────────┐
    │  Running │    │  Disconnected│
    │  (Online)│    │  (Idle)      │
    └──────────┘    └─────────────┘
          │              │
          └──────┬───────┘
                 │
          Error/Timeout
                 ▼
         ┌──────────────┐
         │    Error     │
         │   (Stalled)  │
         └──────────────┘
```

### Stream Types

**Live Streams**
- Real-time input (cameras, encoders)
- Continuous operation
- Optional DVR recording

**VOD Streams**
- File-based input
- Playback on demand
- No recording needed

**Published Streams**
- Accept incoming video via RTMP/WebRTC
- Dynamic configuration
- Temporary or long-lived

## Configuration Files

### Main Configuration File
```
Location: /etc/flussonic/flussonic.conf
Syntax: Erlang-style key-value pairs
Requires: Reload or restart after edit
```

### Configuration Structure
```
# Global settings
edit_auth admin password;
http 80;
https 443;

# Global DVR storage (must define before referencing)
dvr my_storage {
  root /mnt/archive;
}

# VOD locations
vod movies {
  location /storage/movies;
}

# Streams
stream camera1 {
  input rtsp://camera:554/stream;
  protocols hls dash;
  dvr @my_storage 7d;
}
```

Note: Flussonic config syntax does NOT use semicolons at the end of block closings.
Top-level directives use semicolons; blocks use curly braces.

### Hot Reload
Some changes don't require full restart:
```bash
service flussonic reload
```

### Configuration Validation
```bash
# Check syntax
/opt/flussonic/bin/validate_config /etc/flussonic/flussonic.conf
```

## API Basics

### REST API Overview
```
Base URL: http://FLUSSONIC-IP/streamer/api/v3/
```

### Common Endpoints

**Streams** (uses PUT for create AND update — upsert pattern, NOT POST)
```bash
GET    /streamer/api/v3/streams                 # List all streams
GET    /streamer/api/v3/streams/{name}          # Get stream details
PUT    /streamer/api/v3/streams/{name}          # Create or update stream
DELETE /streamer/api/v3/streams/{name}          # Delete stream
POST   /streamer/api/v3/streams/{name}/stop     # Stop stream
```

**Monitoring & Stats**
```bash
GET /streamer/api/v3/config/stats               # Server stats (CPU, RAM, disk, GPU)
GET /streamer/api/v3/monitoring/liveness         # Health check
GET /streamer/api/v3/monitoring/readiness        # Readiness check
```

**DVR**
```bash
GET /streamer/api/v3/dvrs                        # List DVR configs
GET /streamer/api/v3/streams/{name}/dvr/ranges   # Recording ranges
```

**VOD**
```bash
GET /streamer/api/v3/vods                        # List VOD locations
GET /streamer/api/v3/vods/{prefix}               # VOD details
```

### Authentication
```bash
# Uses edit_auth credentials from flussonic.conf
curl -u username:password http://server/streamer/api/v3/streams
```

### Response Format (Streams List)
```json
{
  "server_id": "abc-123",
  "streams": [
    {
      "name": "camera1",
      "stats": {
        "status": "running",
        "alive": true,
        "bitrate": 5000,
        "input": {
          "bytes": 123456,
          "proto": "rtsp",
          "errors": 0
        },
        "online_clients": 5,
        "running_transcoder": true
      },
      "inputs": [{"url": "rtsp://camera:554/stream"}]
    }
  ],
  "estimated_count": 1
}
```

## Logging & Monitoring

### Log Location
```
/var/log/flussonic/flussonic.log
```

### Log Levels
```bash
# View live logs
tail -f /var/log/flussonic/flussonic.log

# Filter by component
grep "stream\|transcode\|hls" /var/log/flussonic/flussonic.log

# Count errors
grep -c "error\|ERROR" /var/log/flussonic/flussonic.log
```

### Common Log Patterns
```
Stream not initialized         # Stream config issue
Input disconnected             # Source unavailable
Transcoder failed              # Encoding error
DVR error                      # Recording issue
API request failed             # API error
```

### Monitoring
```bash
# Server stats (CPU, memory, disk, GPU)
curl -u user:pass http://server/streamer/api/v3/config/stats | jq '.'

# CPU/memory (via system)
ps aux | grep flussonic

# Network connections
netstat -tnp | grep flussonic

# Disk usage
du -sh /var/log/flussonic
du -sh /archive/*
```

## Performance Concepts

### Bitrate and Bandwidth
- Stream bitrate: kbps rate of video
- Network bandwidth: total available capacity
- Formula: bitrate × viewers ≤ available bandwidth

### Latency Components
```
Total Latency = Capture + Network + Processing + Buffering + Display
                (100ms) + (100ms) + (100-500ms) + (1-10s)   + (100ms)
                = 1-11 seconds typical
```

**Reduction strategies:**
- Use low-latency protocols (WebRTC, SRT)
- Reduce segment duration
- Lower buffer sizes
- GPU transcoding

### CPU and GPU
- **CPU Transcoding:** Flexible, slower, efficient for low bitrates
- **GPU Transcoding:** Fast, power-efficient, multiple streams
- **Hardware Coder:** Dedicated, high throughput

### Concurrent Connections
- Depends on: bitrate, CPU, disk I/O, network
- Monitor: active connections in API stats
- Scaling: clustering for 5000+

## Common Workflows

### Basic Live Streaming
```
1. Configure stream with RTMP input
2. Enable HLS output
3. Get playback URL
4. Embed in player
```

### Ingest → Transcode → Deliver
```
1. Ingest RTSP from camera
2. Transcode to H.264/AAC
3. Create HLS variants (3 profiles)
4. Record to DVR
5. Deliver via HLS to clients
```

### Video Restreaming to YouTube
```
1. Ingest stream (WebRTC, RTMP, RTSP)
2. Transcode with HEVC
3. Push via Enhanced RTMP to YouTube
4. Monitor push status
```

### Multi-Server Clustering
```
1. Configure cluster_key and peers
2. Ingest on primary server
3. Pull on secondary servers
4. Distribute viewer load
5. Automatic failover
```

### VOD Delivery
```
1. Create VOD location
2. Upload MP4 files
3. Files auto-available in HLS/DASH
4. Optional: Transcode for quality
5. Serve via CDN
```

### Live → Archive → VOD
```
1. Ingest live stream
2. Enable DVR recording
3. After broadcast, archive segments
4. Make available as VOD
5. Create catchup window
```

## Important Concepts

### Keyframe (I-frame)
- Critical for seeking/switching
- Appears every 1-5 seconds
- Larger file size than other frames
- Essential for HLS/DASH adaptive switching

### Group of Pictures (GOP)
- Sequence between keyframes
- Affects compression and seeking
- Typical: 30-150 frames (1-5 seconds)
- Must match across profiles in ABR

### Manifest
- Playlist file (M3U8 for HLS, MPD for DASH)
- Describes segments and properties
- Updated periodically (live)
- Points to actual media files

### Segment
- Fixed-duration media chunk (typically 6 seconds)
- Enables adaptive bitrate switching
- Packaged in container (TS, MP4, etc.)
- Downloaded/streamed by player

### Profile/Variant
- Specific quality level in adaptive stream
- Defined by: bitrate, resolution, codec
- HLS: separate playlist per profile
- DASH: all in single manifest

