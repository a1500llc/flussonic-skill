# Flussonic Protocols Reference Guide

## Table of Contents
- [Input Protocols](#input-protocols)
- [Output Protocols](#output-protocols)
- [RTMP Protocol](#rtmp-protocol)
- [SRT Protocol](#srt-protocol)
- [HLS Protocol](#hls-protocol)
- [DASH Protocol](#dash-protocol)
- [WebRTC](#webrtc)
- [Protocol Comparison](#protocol-comparison)

## Input Protocols

### Real-Time Protocols

**RTSP (Real Time Streaming Protocol)**
```
input rtsp://192.168.1.100:554/stream;
input rtsp://user:pass@camera:554/stream;
```
- Port: 554 (default)
- Video: H.264, H.265
- Audio: AAC, PCMA, PCMU
- Cameras: Most IP cameras

**RTP (Real-Time Transport Protocol)**
```
input rtp://239.0.0.1:5000;
```
- Port: UDP (variable)
- Video: H.264, H.265
- No audio
- For raw RTP streams

**RTMP (Real-Time Messaging Protocol)**
```
input rtmp://streaming-server:1935/app/stream;
```
- Port: 1935 (default)
- Video: H.264
- Audio: AAC, MP3
- Software encoders (OBS, etc.)

**SRT (Secure Reliable Transport)**
```
# Caller mode (Flussonic pulls from remote SRT source):
input srt://REMOTE-IP:8888 streamid="#!::m=request,r=stream_name";

# Listener mode (accept incoming SRT publish):
# Use publish:// + srt_publish block:
stream my_srt {
  input publish://;
  srt_publish {
    port 9998;
    passphrase 0123456789;
  }
}
```
- Port: configurable per-stream or global
- Codecs: H.264, H.265, VP8/9, AV1
- Encryption via passphrase (10-79 chars)
- Low-latency reliable transport

Note: `srt://listen:PORT` and `mode=caller/listener` are NOT valid Flussonic syntax.

**WebRTC**
```
input publish://;
```
- Port: 8080 (default)
- Codecs: H.264, VP8/9, AV1
- Browser publishing
- Ultra-low latency

### File/HTTP Protocols

**HTTP HLS**
```
input http://source-server:8080/live.m3u8;
```
- Video: H.264, H.265, AV1
- Audio: AAC, EAC3, MP3, AC-3
- Adaptive bitrate

**HTTP/UDP MPEG-TS**
```
input http://source-server:8080/stream.ts;
input udp://239.0.0.1:1234;             # Multicast
```
- Video: H.264, H.265, MPEG-2
- Audio: AAC, EAC3, MP3, MPEG-2

**Multicast UDP**
```
input udp://eth2@239.0.0.1:1234;        # Specify interface
input udp://239.0.0.1:1234/10.100.200.3; # Specify IP
```
- SPTS: Single stream per group
- MPTS: Multiple programs

**File Input**
```
input file:///storage/video.mp4;
input file:///storage/live.m3u8;
```
- Formats: MP4, MOV, M4V
- Codecs: H.264, H.265
- VOD playback

### Special Protocols

**Multicast MPTS**
```
input mpts-dvb://adapter@frequency?program=1001;
```
- DVB satellite/cable input
- Multiple programs

**H.323 (Video Telephony)**
```
input h323://gatekeeper:1300;
```
- Video: H.264, H.265
- Audio: PCMA, PCMU

**Shoutcast/ICEcast**
```
input http://streaming-server:8000/;
```
- Audio only
- Codecs: AAC, MP3

**Fake (Test)**
```
input fake://fake;
```
- Test pattern
- No network I/O

## Output Protocols

### Live Streaming Outputs

**HLS (HTTP Live Streaming)**
```
# Auto-enabled for all streams
http://server/stream/index.m3u8
```

**DASH (MPEG-DASH)**
```
# Auto-enabled for all streams
http://server/stream/manifest.mpd
```

**HTTP MPEG-TS**
```
http://server/stream/index.ts
```

**RTMP Playback**
```
stream mystream { rtmp_play = yes; }
rtmp://server:1935/live/stream
```

**WebRTC Playback**
```
http://server/stream/embed.html
http://server/streamer/api/v3/streams/stream/webrtc
```

**Multicast UDP**
```
push udp://239.0.0.1:1234;
```

## RTMP Protocol

### RTMP vs RTMPS
```
rtmp://server:1935/app/stream         # Plaintext
rtmps://server:443/app/stream         # TLS encrypted
```

### RTMP URL Structure
```
rtmp://[user:password@]host[:port]/app[/instance]/stream
```

**Examples:**
```
rtmp://encoder.example.com:1935/live/camera1
rtmp://user:pass@cdn.example.com:1935/broadcast/event
```

### RTMP/FLV Support
- Containers: FLV
- Video: H.264 (Enhanced RTMP also supports HEVC, AV1)
- Audio: AAC, MP3, PCMU

### Enhanced RTMP
For HEVC/AV1:
```bash
# Requires ffmpeg 6.1.1+
ffmpeg -i input.mp4 -c:v libx265 -c:a copy -f flv \
  rtmp://server:1935/live/stream
```

## SRT Protocol

### SRT Connection Modes

**Caller (Flussonic pulls from remote SRT source):**
```
input srt://REMOTE-IP:8888 streamid="#!::m=request,r=stream_name";
input srt://REMOTE-IP:9999 passphrase=0987654321 streamid="#!::m=request";
```

**Listener (Accept incoming SRT publish):**
```
# Global shared port:
srt_publish {
  port 9998;
  passphrase 0123456789;
}
stream my_srt { input publish://; }

# Per-stream port:
stream my_srt {
  input publish://;
  srt_publish { port 9998; }
}
```

**SRT Output (push):**
```
push srt://destination:1234;
```

**SRT Playback port:**
```
stream my_stream {
  srt_play { port 9300; }
}
```

### SRT Parameters

**Key Parameters:**
- `passphrase=SECRET` — AES encryption (10-79 characters)
- `latency=200` — packet delivery buffer in milliseconds (default 120)
- `enforcedencryption=true` — require matching encryption
- `timeout=N` — data transmission timeout (seconds)
- `connect_timeout=N` — connection timeout (seconds)
- `linger=180` — socket linger time (seconds)

### SRT Streamid Format
```
#!::m=request,r=stream_name    # Pull from remote
#!::m=publish,r=stream_name    # Push to remote
```

Note: `mode=connect/listen` parameter is NOT valid in Flussonic SRT URLs. Use the appropriate config directives instead.

## HLS Protocol

### Segment-Based Streaming
```
Index: #EXTM3U
       #EXT-X-VERSION:3
       #EXT-X-TARGETDURATION:6
       #EXTINF:6.0,
       segment1.ts
       #EXTINF:6.0,
       segment2.ts
```

### Variants (Adaptive)
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000
stream_5000k.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000
stream_2500k.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000
stream_1000k.m3u8
```

### Encryption (EXT-X-KEY)
```
#EXT-X-KEY:METHOD=AES-128,URI="https://server/key.bin"
```

## DASH Protocol

### DASH Manifest (MPD)
```xml
<?xml version="1.0"?>
<MPD>
  <Period>
    <AdaptationSet mimeType="video/mp2t">
      <Representation bandwidth="5000000">
        <BaseURL>segment.ts</BaseURL>
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

### Representation Switching
Player selects quality based on:
- Available bandwidth
- Device capability
- User preference

## WebRTC

### WebRTC Connection
```
# Browser to Flussonic
http://server/streamer/api/v3/streams/stream/webrtc

# Direct embed
http://server/stream/embed.html
```

### WebRTC Codecs
- Video: H.264, VP8, VP9, AV1
- Audio: Opus, PCMA, PCMU

### Low Latency
- Latency: <1 second
- Requires modern browser (Chrome, Firefox, Safari 11+)
- Port: 8080 (default, configurable)

## Protocol Comparison

| Protocol | Latency | Adaptive | Browsers | Reliability | Use Case |
|----------|---------|----------|----------|-------------|----------|
| HLS | 10-30s | Yes | All | TCP reliable | General playback |
| DASH | 10-30s | Yes | Modern | TCP reliable | Enterprise |
| HTTP-TS | 2-5s | No | Apps | TCP reliable | IPTV, set-tops |
| WebRTC | <1s | No | Modern | UDP (lossy) | Live events |
| RTMP | 2-5s | No | Legacy | TCP reliable | Desktop apps |
| SRT | 2-5s | No | Apps | UDP (recoverable) | Reliable low-latency |
| Multicast | 1-3s | No | Special | UDP (lossy) | IPTV networks |

### Selection Criteria
- **Playback:** HLS (universal) or DASH (modern)
- **Low-latency:** WebRTC (<1s) or HTTP-TS (2-5s)
- **Live streaming:** RTMP (ingest) + HLS (playback)
- **IP camera:** RTSP (ingest) + HLS (playback)
- **Reliable low-latency:** SRT
- **IPTV:** MPEG-TS or Multicast
