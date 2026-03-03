# Flussonic Playback & Delivery Guide

## Table of Contents
- [Playback Protocols](#playback-protocols)
- [HLS Configuration](#hls-configuration)
- [DASH Configuration](#dash-configuration)
- [HTTP MPEG-TS](#http-mpeg-ts)
- [RTMP Playback](#rtmp-playback)
- [WebRTC Playback](#webrtc-playback)
- [Player Integration](#player-integration)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)

## Playback Protocols

### Supported Delivery Protocols
- **HLS** (HTTP Live Streaming) - adaptive, wide compatibility
- **DASH** (MPEG-DASH) - adaptive, enterprise
- **HTTP MPEG-TS** - constant bitrate, low latency
- **RTMP** - legacy, low latency
- **WebRTC** - ultra-low latency
- **RTSP** - IP camera playback
- **SRT** - low-latency IP transmission

### Protocol Selection Guide
| Protocol | Latency | Adaptive | Browser | Use Case |
|---|---|---|---|---|
| HLS | 10-30s | Yes | All | General playback |
| DASH | 10-30s | Yes | Modern | Enterprise |
| MPEG-TS | 2-5s | No | VLC/Apps | TV delivery |
| WebRTC | <1s | No | Modern | Live sports/events |
| RTMP | 2-5s | No | Legacy | Desktop apps |

## HLS Configuration

### Default HLS
```
# Automatic HLS generation for any stream
http://FLUSSONIC-IP/mystream/index.m3u8
```

### HLS/DASH Configuration
In Flussonic, protocols and segment parameters are set as top-level stream directives, NOT inside nested blocks:
```
stream mystream {
  input rtsp://camera:554/stream;
  protocols hls dash;
  segment_duration 2;
  segment_count 30;
}
```

**Parameters:**
- `segment_duration N` — segment length in seconds (e.g., 2)
- `segment_count N` — number of segments to keep in playlist (e.g., 30)
- `protocols hls dash` — enable specific output protocols

### Multibitrate HLS (Adaptive)
When transcoding to multiple bitrates, HLS automatically creates a master playlist with all variants:
```
stream hls_adaptive {
  input rtsp://camera:554/stream;
  transcoder vb=5000k size=1920x1080 vb=2500k size=1280x720 vb=1000k size=640x480 ab=128k;
  protocols hls dash;
  segment_duration 2;
  segment_count 30;
}
```

Master playlist includes all bitrates:
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000
stream.m3u8?bitrate=5000k
#EXT-X-STREAM-INF:BANDWIDTH=2500000
stream.m3u8?bitrate=2500k
```

### DRM-Protected HLS/DASH
For DRM protection, use the CPIX standard:
```
stream protected {
  input rtsp://camera:554/stream;
  drm cpix keyserver=http://drm-server/api/drm/cpix?client=myapp&clientToken=SECRET resource_id=protected;
  protocols dash hls;
}
```

## DASH Configuration

### Basic DASH
DASH is enabled via the `protocols` directive:
```
stream mystream {
  input rtsp://camera:554/stream;
  protocols dash;
  segment_duration 4;
}
```

**URL:** `http://FLUSSONIC-IP/mystream/index.mpd`

### Multibitrate DASH
```
stream dash_multi {
  input rtsp://camera:554/stream;
  transcoder vb=5000k size=1920x1080 vb=2500k size=1280x720 vb=1000k size=640x480 ab=128k;
  protocols dash hls;
  segment_duration 2;
}
```

### Key Parameters
- `segment_duration N` — segment length in seconds (top-level stream directive)
- `segment_count N` — segments in playlist

## HTTP MPEG-TS

### Constant Bitrate Streaming
```
http://FLUSSONIC-IP/mystream/index.ts
```

### MPEG-TS Parameters
```
stream ts_stream {
  input rtsp://camera:554/stream;
  mpegts {
    bitrate = 5000;
    segment_duration = 10;
  };
}
```

### Use Cases
- Low latency requirements
- Legacy set-top boxes
- IPTV delivery
- Multicast compatibility

## RTMP Playback

### RTMP Playback URL
```
rtmp://FLUSSONIC-IP:1935/live/mystream
```

### RTMP Configuration
RTMP playback is enabled via the `protocols` directive:
```
stream mystream {
  input rtsp://camera:554/stream;
  protocols hls dash rtmp;
}
```

### RTMP Viewers
```bash
# ffplay
ffplay rtmp://server:1935/live/stream

# VLC
vlc rtmp://server:1935/live/stream
```

## WebRTC Playback

### WebRTC Viewer URL
Via web interface or embed:
```
# Access via UI
http://FLUSSONIC-IP/mystream/embed.html

# WebRTC endpoint
http://FLUSSONIC-IP/streamer/api/v3/streams/mystream/webrtc
```

### WebRTC Configuration
WebRTC playback is enabled via the `protocols` directive:
```
stream webrtc_stream {
  input rtsp://camera:554/stream;
  protocols hls dash webrtc;
}
```

### Low Latency
- Latency: <1 second
- Requires WebRTC-capable device
- Best for live events
- Chrome, Firefox, Safari 11+

## Player Integration

### HLS Player Example (hls.js)
```html
<video id="player" controls width="640" height="480"></video>
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script>
  var video = document.getElementById('player');
  var hls = new Hls();
  hls.loadSource('http://server/stream/index.m3u8');
  hls.attachMedia(video);
</script>
```

### DASH Player Example (dash.js)
```html
<video id="player" controls width="640" height="480"></video>
<script src="https://cdn.dashjs.org/latest/dash.all.min.js"></script>
<script>
  var player = dashjs.MediaPlayer().create();
  player.initialize(document.getElementById('player'),
    'http://server/stream/manifest.mpd', true);
</script>
```

### Native HTML5
```html
<video controls width="640" height="480">
  <source src="http://server/stream/index.ts" type="video/mp2t">
  <source src="http://server/stream/index.m3u8" type="application/vnd.apple.mpegurl">
</video>
```

## Performance

### Segment Size Tuning
```
Duration (s) | Bitrate (kbps) | Segment Size (MB)
2            | 5000           | 1.2
4            | 5000           | 2.4
6            | 5000           | 3.6
10           | 5000           | 6.0
```

### Buffer Strategy
- Small segments: lower latency, more requests
- Large segments: higher latency, fewer requests

### ABR Settings
Adaptive bitrate is automatic when transcoding to multiple quality levels.
Configure transcoder with multiple `vb=` entries and enable protocols:
```
transcoder vb=5000k size=1920x1080 vb=2500k size=1280x720 vb=1000k size=640x360 ab=128k;
protocols hls dash;
```

## Troubleshooting

### Video Not Playing
- Check stream is active: curl -u user:pass http://server/streamer/api/v3/streams
- Verify protocol enabled: UI > Stream > Output
- Test URL directly: `curl http://server/stream/index.m3u8`
- Check firewall/ports

### Playback Stalls
- Increase segment size: `segment_duration = 10`
- Check network bandwidth
- Monitor server CPU/disk
- Verify no transcoding errors

### Latency Issues
- Use HTTP MPEG-TS (2-5s)
- Switch to WebRTC (<1s)
- Reduce segment duration
- Use shorter GOP on encoder

### Adaptive Bitrate Not Working
- Ensure multiple transcoder profiles
- Set `variant = bitrate`
- Check client bandwidth
- Review player ABR settings

### Commands
```bash
# Check playback status
curl -u user:pass http://server/streamer/api/v3/streams/mystream | \
  jq '.outputs'

# View HLS playlist
curl http://localhost/mystream/index.m3u8

# Test MPEG-TS
ffplay http://localhost/mystream/index.ts

# Monitor bitrate
curl -u user:pass http://server/streamer/api/v3/streams?select=name,stats.status,stats.bitrate | jq '.streams'
```
