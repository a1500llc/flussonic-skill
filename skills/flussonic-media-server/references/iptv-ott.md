# Flussonic IPTV & OTT Guide

## Table of Contents
- [IPTV Overview](#iptv-overview)
- [OTT Overview](#ott-overview)
- [M3U Playlist Format](#m3u-playlist-format)
- [EPG (Electronic Program Guide)](#epg-electronic-program-guide)
- [IPTV Channel Configuration](#iptv-channel-configuration)
- [OTT VOD Delivery](#ott-vod-delivery)
- [DRM for IPTV/OTT](#drm-for-iptv-ott)
- [Best Practices](#best-practices)

## IPTV Overview

IPTV (Internet Protocol Television) provides live TV and content over IP networks:
- Set-top box compatible
- Multicast or unicast delivery
- Linear TV channels
- EPG integration
- Middleware support

### Typical IPTV Architecture
```
Content Sources
      |
   Flussonic (Head-end)
      |
  Network (Multicast/Unicast)
      |
   Set-Top Boxes/Apps
```

## OTT Overview

OTT (Over-The-Top) delivers content via internet:
- Browser and app-based
- Adaptive bitrate (HLS/DASH)
- VOD and catch-up TV
- DRM protected
- Global distribution

## M3U Playlist Format

### Basic M3U
```
#EXTM3U
#EXTINF:-1,Channel 1
http://server/stream1/index.m3u8
#EXTINF:-1,Channel 2
http://server/stream2/index.m3u8
```

### M3U with Extended Info
```
#EXTM3U
#EXTINF:-1 tvg-id="ch1" tvg-name="Channel 1" tvg-logo="logo.png" group-title="Movies",Channel 1
http://server/stream1/index.m3u8

#EXTINF:-1 tvg-id="ch2" tvg-name="Channel 2" tvg-logo="logo2.png" group-title="Sports",Channel 2
http://server/stream2/index.m3u8
```

### M3U Attributes
- `tvg-id` - unique channel ID
- `tvg-name` - display name
- `tvg-logo` - logo URL
- `group-title` - category/group
- `duration` - duration in seconds (-1 for live)

### Generate M3U Dynamically
```bash
# Create M3U from API
curl http://localhost/api/v3/streams | jq -r '.[] | 
  "#EXTINF:-1 tvg-id=\"\(.name)\" tvg-name=\"\(.name)\",\(.name)\n" +
  "http://localhost/\(.name)/index.m3u8"' > playlist.m3u
```

## EPG (Electronic Program Guide)

### XMLTV Format
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE tv SYSTEM "xmltv.dtd">
<tv>
  <channel id="ch1">
    <display-name>Channel 1</display-name>
  </channel>
  <programme start="202402251800" stop="202402251900" channel="ch1">
    <title>Program Title</title>
    <description>Program description</description>
    <category>Sports</category>
  </programme>
</tv>
```

### EPG Integration
```
# Configure EPG in Flussonic
epg {
  source = "http://epg-server/xmltv.xml";
  update_interval = 3600;
}
```

### Stream EPG URL
```
stream sports {
  input rtsp://camera:554/stream;
  epg_id = "ch_sports";
}
```

## IPTV Channel Configuration

### Multicast Delivery (Unicast Alternative)
```
# Ingest from source
stream live_channel {
  input rtsp://encoder:554/stream;
}

# Output to multicast
output push://udp://239.0.0.1:1234;
```

**Playback:**
```
vlc udp://@239.0.0.1:1234
```

### Transcoding for Set-Top Box
```
stream set_top_box_stream {
  input rtsp://camera:554/stream;
  
  # Transcode to compatible format
  transcoder vb=2000k size=1280x720 ab=128k;
  
  # Output as MPEG-TS
  output push://udp://239.0.0.1:5000;
}
```

### Server-Side Playlist (Playout)
```
stream playlist_channel {
  input file:///vod/intro.mp4;
  input rtsp://live-source/stream;
  input file:///vod/outro.mp4;
}
```

**Sequence:** Intro video > Live stream > Outro video

## OTT VOD Delivery

### VOD Location Setup
```
vod movies {
  location = /storage/movies;
}

vod shows {
  location = /storage/shows;
}
```

### Multibitrate VOD
For OTT, provide multiple quality options:
```
# API to get VOD manifest
curl http://localhost/api/v3/vods/movies
```

### VOD with DRM
```
vod protected {
  location = /storage/drm-content;
  drm widevine {
    license_url = "https://license-server/widevine";
  };
}
```

## DRM for IPTV/OTT

### IPTV (Set-Top Box) DRM
```
stream protected_iptv {
  input rtsp://source/stream;
  dvb_ci_ca;  # DVB Common Interface
}
```

### OTT Streaming DRM
```
stream protected_ott {
  input rtsp://source/stream;
  
  # Multiple DRM for device compatibility
  drm widevine {
    license_url = "https://license-server/widevine";
  };
  
  drm playready {
    license_url = "https://license-server/playready";
  };
  
  drm fairplay {
    license_url = "https://license-server/fairplay";
  };
}
```

## Best Practices

### IPTV Considerations
1. **Multicast:** For LAN delivery (efficient)
   - Register multicast addresses
   - Configure switches/routers
   - Limited to local network

2. **Unicast:** For WAN (internet)
   - More compatible
   - Higher bandwidth
   - Scalable globally

3. **Set-Top Box Compatibility**
   - Test with actual STBs
   - Verify MPEG-TS support
   - Check codec support (H.264 minimum)

### OTT Best Practices
1. **Adaptive Bitrate**
   ```
   stream ott_adaptive {
     input rtsp://source/stream;
     transcoder vb=5000k size=1920x1080;
     transcoder vb=2500k size=1280x720;
     transcoder vb=1000k size=640x480;
     hls { variant = bitrate; };
   }
   ```

2. **CDN Optimization**
   - Use clustering for scale
   - Place servers geographically
   - Monitor network latency

3. **Quality Levels**
   - 4K: 4000-8000 kbps
   - 1080p: 2500-5000 kbps
   - 720p: 1000-2500 kbps
   - Mobile: 500-1000 kbps

4. **Segment Configuration**
   ```
   hls {
     segment_duration = 6;
     window_size = 5;
   }
   ```

### Catch-Up TV
Archive + VOD:
```
stream catchup {
  input rtsp://source/stream;
  dvr /archive/channel1;
  hls;
}
```

**Playback:**
```
# Live
http://server/catchup/index.m3u8

# Archive (24 hours)
http://server/catchup/archive-YYYYMMDD/HHMMSS.m3u8
```

### Monitoring
```bash
# Stream health
curl http://localhost/api/v3/streams | jq '.[] | {name, bitrate, viewers}'

# Check EPG
curl http://localhost/api/v3/epg

# VOD inventory
curl http://localhost/api/v3/vods
```
