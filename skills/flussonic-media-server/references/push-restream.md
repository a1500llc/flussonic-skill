# Flussonic Push & Restreaming Guide

## Table of Contents
- [Push Overview](#push-overview)
- [Configuration](#configuration)
- [RTMP Restreaming](#rtmp-restreaming)
- [SRT Push](#srt-push)
- [HTTP/HLS Push](#httphls-push)
- [Social Platform Integration](#social-platform-integration)
- [Advanced Scenarios](#advanced-scenarios)
- [Troubleshooting](#troubleshooting)

## Push Overview

Push/restreaming allows sending streams from Flussonic to external platforms:
- RTMP servers (Wowza, nginx-rtmp, etc.)
- CDN endpoints (Akamai, Cloudflare, etc.)
- Social platforms (YouTube, Facebook, Twitch)
- Remote Flussonic servers
- Video platforms (OBS, streaming tools)

## Configuration

### Basic Push
```
stream mystream {
  input rtsp://camera:554/stream;
  output push://rtmp://destination-server:1935/app/stream;
}
```

### Multiple Destinations
```
stream multicast_push {
  input rtsp://camera:554/stream;
  output push://rtmp://server1:1935/app/stream;
  output push://rtmp://server2:1935/app/stream;
  output push://rtmp://server3:1935/app/stream;
}
```

### Push Parameters
```
output push://rtmp://server:1935/app/stream \
  retry_timeout=10 \
  retry_count=5 \
  timeout=30 \
  buffer=1024 \
  backlog=unlimited;
```

**Parameters:**
- `retry_timeout` - seconds between retries
- `retry_count` - max retry attempts (-1 = infinite)
- `timeout` - connection timeout in seconds
- `buffer` - output buffer in KB
- `backlog` - queue management

## RTMP Restreaming

### Standard RTMP Push
```
stream live {
  input rtsp://camera:554/stream;
  output push://rtmp://rtmp-server.com:1935/live/camera1;
}
```

### RTMPS (Secure RTMP)
```
stream secure_push {
  input rtsp://camera:554/stream;
  output push://rtmps://secure-server.com:443/app/stream;
}
```

### With Credentials
```
output push://rtmp://user:password@server:1935/app/stream;
```

### Enhanced RTMP (HEVC/AV1)
```
stream high_quality {
  input webrtc://;
  transcoder vcodec=hevc vb=5000k preset=fast;
  output push://rtmp://youtube-server/live/stream-key;
}
```

## SRT Push

### Basic SRT Push
```
stream srt_push {
  input rtsp://camera:554/stream;
  output push://srt://remote-server:1234;
}
```

### SRT with Options
```
output push://srt://remote-server:1234?passphrase=secret&latency=200&mode=connect;
```

**SRT Options:**
- `passphrase` - encryption password
- `latency` - in milliseconds
- `mode` - connect/listen
- `congest_ctrl` - live/file

## HTTP/HLS Push

### HLS Segment Upload
```
stream hls_push {
  input rtsp://camera:554/stream;
  output push://http://cdn-server:8080/upload/;
}
```

### HTTP PUT
```
output push://http://api-server/streams/mystream/segment?token=xyz;
```

## Social Platform Integration

### YouTube Live Streaming
```
stream youtube_live {
  input rtsp://camera:554/stream;
  transcoder vb=5000k preset=fast;
  output push://rtmp://a.rtmp.youtube.com/live2/YOUR_STREAM_KEY;
}
```

**Get Stream Key:**
1. YouTube Studio > Content > Live
2. Copy RTMP Server URL and Stream Key
3. Combine: `rtmp://server/live2/key`

### Facebook Live
```
stream facebook_live {
  input rtsp://camera:554/stream;
  output push://rtmps://live-api-s.facebook.com:443/rtmp/YOUR_STREAM_KEY;
}
```

### Twitch Live
```
stream twitch_live {
  input rtsp://camera:554/stream;
  output push://rtmp://live.twitch.tv/app/YOUR_STREAM_KEY;
}
```

### Multi-Platform
```
stream multi_platform {
  input rtsp://camera:554/stream;
  output push://rtmp://a.rtmp.youtube.com/live2/yt_key;
  output push://rtmps://live-api-s.facebook.com:443/rtmp/fb_key;
  output push://rtmp://live.twitch.tv/app/twitch_key;
}
```

## Advanced Scenarios

### Transcode Before Push
```
stream optimized_push {
  input rtsp://camera:554/stream;
  
  # Transcode for better quality
  transcoder vb=2500k size=1280x720 preset=fast ab=128k;
  
  # Push to CDN
  output push://rtmp://cdn-server:1935/app/stream;
  
  # Also provide HLS locally
  hls;
}
```

### Conditional Push (Template)
```
template live- {
  input rtsp://camera:554/stream;
  
  output push://rtmp://backup-server:1935/app/$name$;
  
  hls;
}
```

### Push with Failover
```
output push://rtmp://primary:1935/app/stream retry_count=-1 retry_timeout=5;
output push://rtmp://secondary:1935/app/stream retry_count=-1 retry_timeout=10;
```

**Behavior:**
- Tries primary first
- Falls back to secondary on failure
- Retries indefinitely

### Multicasting
```
stream multicast_out {
  input rtsp://camera:554/stream;
  output push://udp://239.0.0.1:1234;
}
```

## Troubleshooting

### Push Connection Fails
- **Check URL:** Verify server address, port, path
- **Network:** Ping destination server
- **Firewall:** Verify outbound port open
- **Credentials:** Check username/password

```bash
# Test connectivity
nc -zv destination-server 1935

# Check DNS resolution
nslookup destination-server

# Verify firewall
iptables -L | grep 1935
```

### Push Drops/Reconnects
- Check network stability: ping remote server continuously
- Increase retry timeout: `retry_timeout=30`
- Monitor bitrate: may exceed network capacity
- Check server logs for disconnect reason

### High Latency on Push
- Use `buffer=0` for lowest latency
- Check network RTT: `ping -i 0.2 remote-server`
- Reduce segment duration if HLS
- Consider SRT with lower latency

### Audio/Video Sync Issues
- Ensure input stream has sync audio/video
- Set same FPS for all outputs
- Check transcoder: `fps=video`
- Review logs for dropped frames

### Platform-Specific

**YouTube errors:**
```
- 403 Forbidden: Invalid stream key
- 400 Bad Request: HEVC not supported with certain codec
- 503 Service Unavailable: Try again later
```

**Facebook errors:**
```
- Authentication failed: Check stream key
- Not authorized: Verify page permissions
- Stream ended: Session timeout
```

**Twitch errors:**
```
- Invalid RTMP URL: Check server and port
- Authentication failed: Invalid stream key
- Slow upload: Network bandwidth issue
```

### Commands
```bash
# Check push status
curl http://localhost/api/v3/streams/mystream | jq '.outputs'

# Monitor push connections
netstat -tnp | grep push

# View push logs
tail -f /var/log/flussonic/flussonic.log | grep -i push

# Test RTMP server
ffmpeg -f lavfi -i testsrc=size=320x240:duration=5 -f flv \
  rtmp://destination:1935/app/stream
```

### Performance Tips
- Use hardware transcoding for multiple pushes
- Limit bitrate to network capacity
- Monitor CPU during multi-push scenarios
- Use separate streams for different qualities
- Consider clustering for high-volume push
