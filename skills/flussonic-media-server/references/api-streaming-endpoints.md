# Flussonic API Streaming - Playback & Delivery Endpoints

## Table of Contents
- [Overview](#overview)
- [HLS Streaming](#hls-streaming)
- [DASH Streaming](#dash-streaming)
- [MPEG-TS Streaming](#mpeg-ts-streaming)
- [DVR/Archive Playback](#dvrarchive-playback)
- [Rewind & Timeshift](#rewind--timeshift)
- [Manifest Formats](#manifest-formats)
- [Query Parameters](#query-parameters)

## Overview

**Base URL:** `http://server/`
**No authentication:** Streams accessible by default (unless auth enabled)
**Dynamic:** Endpoints work for any stream or VOD location
**Formats:** Multiple protocol and container options

### Stream URL Pattern
```
http://server/{stream_name}/{format}.{extension}
http://server/{stream_name}/{protocol}-{variant}.{extension}
```

## HLS Streaming

### Live HLS (MPEG-TS variant)
```
GET /{stream}/index.ts.m3u8
```
- Segments in MPEG-TS container
- Works on older devices
- Wide compatibility

**Example:**
```
http://server/camera1/index.ts.m3u8
```

### Live HLS (fMP4 variant)
```
GET /{stream}/video.m3u8
GET /{stream}/index.mp4.m3u8
```
- Segments in ISO Base Media File Format (fMP4)
- Modern browsers and apps
- Better codec support (HEVC, AV1)

**Example:**
```
http://server/camera1/video.m3u8
```

### Master Playlist (Multiple Bitrates)
```
GET /{stream}/index.m3u8
```
- References all available bitrate variants
- Player selects based on bandwidth
- Generated automatically for multibitrate streams

**Manifest content:**
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000
stream_5000k.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000
stream_2500k.m3u8
```

### Variant-Specific Playlist
```
GET /{stream}/index.m3u8?bitrate=2500k
GET /{stream}/index.m3u8?index=1
```
- Get specific quality variant
- Useful for playback control

## DASH Streaming

### DASH Manifest (MPD)
```
GET /{stream}/manifest.mpd
```
- MPEG-DASH dynamic manifest
- Contains all quality levels
- Real-time updates for live streams

**Usage:**
```
http://server/camera1/manifest.mpd
```

### DASH Static Segments
```
GET /{stream}/manifest.mpd?static=1
```
- Static manifest (for VOD)
- No real-time updates

## MPEG-TS Streaming

### Raw MPEG-TS
```
GET /{stream}/index.ts
```
- Continuous MPEG-TS stream
- Low latency (2-5 seconds)
- Constant bitrate
- Traditional set-top box format

**Usage:**
```
http://server/camera1/index.ts
```

### MPEG-TS Progressive Download
```
GET /{stream}/index.ts?start=10&duration=30
```
- Download specific duration
- Client controls playback timing
- Supports seeking

**Parameters:**
- `start` - start position (seconds from beginning)
- `duration` - duration to download (seconds)

## DVR/Archive Playback

### Archive by Date and Duration
```
GET /{stream}/archive-{YYYYMMDD}-{duration}.{format}
```
- Archive playback of recorded segments
- Date-based access
- Duration in seconds

**Example:**
```
http://server/camera1/archive-20240225-7200.ts
http://server/camera1/archive-20240225-7200.ts.m3u8
```

### Archive Timepoint Specification
```
GET /{stream}/archive-{YYYYMMDD}/{HHMMSS}.m3u8
```
- Direct timestamp format
- More precise than duration

**Example:**
```
http://server/camera1/archive-20240225/103000.m3u8
```

### List Available Archive
```
GET /api/v3/streams/{stream}/segments
Query: from=ISO8601&to=ISO8601
```
- Returns available recording windows
- JSON response with timestamps

**Example:**
```
curl 'http://server/api/v3/streams/camera1/segments?from=2024-02-25T10:00:00Z&to=2024-02-25T12:00:00Z'
```

## Rewind & Timeshift

### Rewind (Go Back N Seconds)
```
GET /{stream}/rewind-{seconds}.{format}
```
- Playback from N seconds ago
- For live streams with DVR enabled
- Continuous time-shift playback

**Example:**
```
http://server/camera1/rewind-300.ts.m3u8    (rewind 5 minutes)
http://server/camera1/rewind-3600.ts.m3u8   (rewind 1 hour)
```

### Absolute Timeshift
```
GET /{stream}/timeshift_abs-{YYYYMMDDTHHMMSSZ}.ts.m3u8
```
- Playback from absolute timestamp
- Start from specific date/time
- Full ISO 8601 format

**Example:**
```
http://server/camera1/timeshift_abs-20240225T103000Z.ts.m3u8
```

### Relative Timeshift
```
GET /{stream}/timeshift_rel-{delay_seconds}.ts.m3u8
```
- Playback with N-second delay
- Useful for safety delay in live broadcast

**Example:**
```
http://server/camera1/timeshift_rel-10.ts.m3u8  (10 second delay)
```

## Manifest Formats

### JSON Manifest
```
GET /{stream}/index.json
GET /{stream}/archive-{YYYYMMDD}-{duration}.json
GET /{stream}/rewind-{seconds}.json
```
- Machine-readable manifest
- Contains segment metadata
- Useful for API-driven players

**Response:**
```json
{
  "duration": 86400,
  "segments": [
    {
      "duration": 6.0,
      "offset": 0,
      "uri": "segment1.ts"
    }
  ]
}
```

### M3U8 (HLS Playlist)
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXTINF:6.0,
segment1.ts
#EXTINF:6.0,
segment2.ts
```

### MPD (DASH Manifest)
```xml
<?xml version="1.0"?>
<MPD type="dynamic" publishTime="...">
  <Period>
    <AdaptationSet mimeType="video/mp2t">
      <Representation bandwidth="5000000">
        ...
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

## Query Parameters

### Stream Selection
| Parameter | Purpose | Example |
|-----------|---------|---------|
| `bitrate` | Select quality variant | `?bitrate=2500k` |
| `index` | Variant index (0-based) | `?index=1` |

### Token/Authentication
| Parameter | Purpose | Example |
|-----------|---------|---------|
| `token` | Authentication token | `?token=abc123` |
| `secure_token` | Secure link token | `?secure_token=xyz` |

### Segment Control
| Parameter | Purpose | Example |
|-----------|---------|---------|
| `start` | Start position (seconds) | `?start=10` |
| `duration` | Duration (seconds) | `?duration=30` |

### Format Control
| Parameter | Purpose | Example |
|-----------|---------|---------|
| `static` | Static manifest (DASH) | `?static=1` |
| `format` | Force format | `?format=json` |

### Example URLs with Parameters
```
# Token-protected HLS
http://server/stream/index.m3u8?token=abc123

# Specific quality
http://server/stream/index.m3u8?bitrate=1000k

# Archive with token
http://server/stream/archive-20240225-3600.ts.m3u8?token=xyz

# Rewind HLS
http://server/stream/rewind-600.ts.m3u8  (10 min rewind)
```

## VOD Playback

### VOD File Direct
```
GET /{vod_name}/{file_path}.mp4
GET /{vod_name}/{file_path}/index.m3u8
GET /{vod_name}/{file_path}/manifest.mpd
```

**Examples:**
```
http://server/movies/BigBuckBunny.mp4
http://server/movies/BigBuckBunny.mp4/index.m3u8
http://server/shows/Season1/Episode1.mp4/manifest.mpd
```

## Embedded Player

### Flussonic Embed
```
GET /{stream}/embed.html
```
- Built-in HTML5 player
- Auto-detects browser capabilities
- No additional configuration needed

**Usage:**
```html
<iframe src="http://server/camera1/embed.html"
  width="640" height="480" frameborder="0" allowfullscreen>
</iframe>
```

## Common Usage Patterns

### All Formats for Stream
```bash
# HLS
http://server/stream/index.m3u8
http://server/stream/index.ts.m3u8

# DASH
http://server/stream/manifest.mpd

# Raw MPEG-TS
http://server/stream/index.ts

# JSON
http://server/stream/index.json
```

### Archive Playback
```bash
# List available archive
curl http://server/api/v3/streams/stream/segments

# Play specific date
http://server/stream/archive-20240225-3600.ts.m3u8

# Play from timestamp
http://server/stream/archive-20240225/140000.m3u8
```

### Adaptive Streaming
```bash
# Master playlist (all bitrates)
http://server/stream/index.m3u8

# Specific bitrate
http://server/stream/index.m3u8?bitrate=2500k
```

### Low-Latency Playback
```bash
# MPEG-TS (lowest latency)
http://server/stream/index.ts

# MPEG-TS with rewind
http://server/stream/rewind-30.ts
```
