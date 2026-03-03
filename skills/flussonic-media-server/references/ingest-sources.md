# Flussonic Ingest Sources Guide

## Table of Contents
- [Supported Protocols](#supported-protocols)
- [Source Requirements](#source-requirements)
- [RTMP Ingest](#rtmp-ingest)
- [RTSP/RTP Ingest](#rtsptprtp-ingest)
- [SRT Ingest](#srt-ingest)
- [HTTP/Multicast Ingest](#httpmulticast-ingest)
- [WebRTC Publishing](#webrtc-publishing)
- [IP Camera Ingest](#ip-camera-ingest)
- [Stream Validation](#stream-validation)

## Supported Protocols

### Live Streaming Protocols

Protocol | Container | Video Codec | Audio Codec
---|---|---|---
RTMP | FLV | H.264 | AAC, MP3, PCMU
RTSP | - | H.264, H.265 (HEVC) | AAC, PCMA, PCMU
RTP | - | H.264, H.265 (HEVC) | -
HLS | MPEG-TS | H.264, H.265, AV1 | AAC, EAC3, MP3, AC-3
SRT | container-agnostic | H.264, H.265, MPEG-2, VP8, VP9, AV1 | AAC, EAC3, MP3, AC-3, Opus
HTTP/UDP | MPEG-TS | H.264, H.265, MPEG-2 | AAC, EAC3, MP3, MPEG-2
WebRTC | - | H.264, VP8, VP9, AV1 | Opus, PCMA, PCMU
H.323 | - | H.264, H.265 | PCMA, PCMU
Multicast | MPEG-TS | H.264, H.265, MPEG-2 | AAC, EAC3, MP3

### File Formats (VOD)

Container | Video Codec | Audio Codec
---|---|---
MP4 (.mp4, .mov, .m4v, .3gp) | H.264, H.265 (HEVC) | MP3, AAC

## Source Requirements

### Minimum FPS
- Minimum 10 FPS for stream to be active
- Recommended 15+ FPS for stability
- Some cameras need FPS increased to avoid fragmented output

### Unsupported Streams
If source has unsupported codecs/containers, use transcoding (see transcoding guide).

### Network Tuning for UDP
For multicast/UDP sources, increase Linux network buffers:
- Recommended: 16MB for HD video
- Configure in /etc/sysctl.conf

## RTMP Ingest

### Basic Configuration
```
stream mystream {
  input rtmp://camera-ip:1935/live/stream;
}
```

### Publishing (Static Stream)
```
stream static_name {
  input publish://;
}
```

### Publishing (Dynamic Template)
```
template live-prefix {
  input publish://;
  prefix prefix;
}
```

**Publish URL:** `rtmp://FLUSSONIC-IP:1935/prefix/STREAM_NAME`

### Enhanced RTMP (HEVC/AV1)
Flussonic supports Enhanced RTMP for pushing high-quality codecs to platforms like YouTube. Requires ffmpeg 6.1.1+:
```bash
ffmpeg -i input.mp4 -c:v libx265 -preset fast -c:a copy -f flv \
  rtmp://localhost:1935/static/published
```

## RTSP/RTP Ingest

### RTSP (Real Time Streaming Protocol)
```
stream camera1 {
  input rtsp://192.168.1.100:554/stream1;
}
```

### With Authentication
```
stream camera1 {
  input rtsp://user:password@192.168.1.100:554/stream1;
}
```

### RTP (Raw Video)
```
stream rtp_stream {
  input rtp://239.0.0.1:5000;
}
```

## SRT Ingest

### SRT Caller Mode (Flussonic pulls from remote SRT source)
```
stream srt_pull {
  input srt://REMOTE-IP:8888 streamid="#!::m=request,r=stream_name";
}
```

With passphrase:
```
stream srt_pull_encrypted {
  input srt://REMOTE-IP:9999 passphrase=0987654321 streamid="#!::m=request";
}
```

### SRT Listener Mode (Accept incoming SRT publish)

**Global shared port (all streams share one port):**
```
srt_publish {
  port 9998;
  passphrase 0123456789;
}

stream my_srt_stream {
  input publish://;
}
```

**Per-stream dedicated port:**
```
stream my_srt_stream {
  input publish://;
  srt_publish {
    port 9998;
    passphrase 0123456789;
  }
}
```

**Streamid format for SRT:** `#!::` followed by key=value pairs:
- `r=STREAM_NAME` — stream name
- `m=publish` — for sending to Flussonic
- `m=request` — for pulling from Flussonic

**Common SRT Parameters:**
- `passphrase` — encryption password (10-79 characters)
- `latency` — packet delivery delay buffer in ms (default 120ms)
- `enforcedencryption` — require matching encryption (default true)
- `timeout` — data transmission timeout in seconds
- `connect_timeout` — connection timeout in seconds

Note: The syntax `srt://listen:PORT` does NOT work in Flussonic. Use `input publish://;` with `srt_publish {}` or `srt PORT;` for listener mode.

## HTTP/Multicast Ingest

### Multicast SPTS (Single Program Transport Stream)
```
stream multicast1 {
  input udp://239.0.0.1:1234;
}
```

### Specify Network Interface
By interface name:
```
stream multicast1 {
  input udp://eth2@239.0.0.1:1234;
}
```

By interface IP:
```
stream multicast1 {
  input udp://239.0.0.1:1234/10.100.200.3;
}
```

### Multicast with IGMPv3 Source Filtering
Real-world multicast often uses IGMPv3 with `sources=` parameter:
```
stream multicast_igmpv3 {
  input udp://239.77.16.88:8999/172.17.202.18?sources=10.77.22.88&buffer_size=8M&pkt_size=1316;
}
```

Parameters:
- `sources=IP` — IGMPv3 source filter (SSM)
- `buffer_size=8M` — receive buffer size
- `pkt_size=1316` — expected packet size
- `cc_check=repeat` — continuity counter check mode
- `flushpcr=1` — flush on PCR

### Multicast Troubleshooting
```bash
# Remove firewall rules
iptables -F

# Disable rp_filter on interface
sysctl -w 'net.ipv4.conf.eth0.rp_filter=0'
```

### MPTS (Multiprogram Transport Stream)
Use protocol-specific options instead of udp://

### HTTP Source
```
stream hls_input {
  input http://source-server:8080/live/stream.m3u8;
}
```

## WebRTC Publishing

### Configure WebRTC Port
In UI: Config > Settings > Listeners > set port for WebRTC (default 8080)

### Publish Stream
```
stream webrtc_published {
  input publish://;
}
```

### Adaptive Bitrate (ABR) Settings
```
stream webrtc_stream {
  input publish:// abr_loss_lower=2 abr_loss_upper=10 abr_mode=1 \
    abr_stepdown=50 frames_timeout=1 abr_max_bitrate=2200 \
    min_bitrate=500 output_audio=aac priority=0 source_timeout=5;
}
```

**ABR Parameters:**
- `abr_loss_lower` - increase bitrate when losses below this %
- `abr_loss_upper` - decrease bitrate when losses above this %
- `abr_stepdown`/`abr_stepup` - adjustment step size (kbps)
- `abr_max_bitrate` - maximum bitrate for publication
- `min_bitrate` - minimum bitrate for publication
- `abr_cycles` - cycles before bitrate considered optimal (0=always adjust)

## IP Camera Ingest

### Generic IP Camera
```
stream ipcam {
  input rtsp://admin:password@192.168.1.100:554/stream;
}
```

### Common Camera Ports
- RTSP: 554, 8554
- HTTP: 80, 8080
- RTMP: 1935

### Camera with Audio Issues
If camera has audio but it's problematic:
1. Go to Input tab
2. Click Options next to source URL
3. Select output audio encoding (AAC, MP3, etc.)

### Single Connection Recommendation
Limit camera to 1 concurrent connection to Flussonic to prevent:
- Camera overheating
- Freezing
- Performance degradation

## Stream Validation

### Check Stream Health
1. Stream is active (connected)
2. Uptime grows continuously
3. Input bitrate matches expected (varies 1-10 Mbps)

### Monitor in UI
- Media > Streams > [stream name] > Logs
- Watch for connection errors
- Check codec and bitrate details

### Command Line Check
```bash
# Check Flussonic service status
service flussonic status

# View logs
tail -f /var/log/flussonic/flussonic.log

# Check active streams
curl -s http://server/streamer/api/v3/streams | jq .
```

### Common Issues
- **Stream disconnects:** Check source, network, firewall
- **No audio:** Verify codec support, may need transcoding
- **High latency:** Check network, SRT latency settings
- **Packet loss:** Increase UDP buffer, check network quality
- **Codec mismatch:** Check source format vs requirements
