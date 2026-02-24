# Flussonic Transcoding Guide

## Table of Contents
- [Transcoder Basics](#transcoder-basics)
- [Configuration Syntax](#configuration-syntax)
- [Hardware Acceleration](#hardware-acceleration)
- [Output Profiles](#output-profiles)
- [Video Codecs](#video-codecs)
- [Audio Handling](#audio-handling)
- [Advanced Features](#advanced-features)
- [Flussonic Coder](#flussonic-coder)
- [Troubleshooting](#troubleshooting)

## Transcoder Basics

### When to Use Transcoding
- Input stream has unsupported codec
- Need multiple quality levels (ABR)
- Require format conversion (MPEG-2 to H.264)
- Deinterlace video (satellite/DVB sources)
- Add watermarks, text, or logos
- Convert audio codecs

### Built-in Transcoder
Flussonic includes a software transcoder. For GPU acceleration, use:
- Nvidia NVENC (recommended)
- Intel Quick Sync (QSV)
- Or hardware: Flussonic Coder

## Configuration Syntax

### Basic Transcoding
```
stream mystream {
  input rtsp://camera:554/stream;
  transcoder vb=2000k ab=128k;
}
```

### Multiple Profiles (Multibitrate)
```
stream hd {
  input rtsp://camera:554/stream;
  transcoder vb=5000k size=1920x1080 ab=192k preset=fast;
  transcoder vb=2500k size=1280x720 ab=128k preset=medium;
  transcoder vb=1000k size=640x480 ab=64k preset=medium;
}
```

### Key Parameters

**Bitrate:**
- `vb=VALUE` - video bitrate (e.g., 2000k, 5M)
- `ab=VALUE` - audio bitrate (e.g., 128k, 192k)

**Size/Resolution:**
- `size=WIDTHxHEIGHT` - output resolution (e.g., 1920x1080)
- `-2` for automatic scaling (keeps aspect ratio)
  Example: `size=1280x-2`

**Encoding:**
- `preset=VALUE` - encoding speed (ultrafast, fast, medium, slow)
  - ultrafast: lowest quality, lowest CPU
  - slow: highest quality, highest CPU

**Deinterlacing:**
- `deinterlace=yadif` - best quality (CPU intensive)
- `deinterlace=bwdif` - better quality
- `deinterlace=w3fdif` - highest quality (very slow)
- `deinterlace=simple` - basic deinterlace

**Framerate:**
- `fps=VALUE` - output FPS (e.g., fps=25, fps=30)
- `fps=video` - match input FPS

## Hardware Acceleration

### Nvidia NVENC
```
stream gpu_stream {
  input rtsp://camera:554/stream;
  transcoder hw=nvidia vb=2000k ab=128k preset=fast;
}
```

**GPU Memory:** Each GPU can handle ~10-20 HD transcodes depending on model

### Intel Quick Sync (QSV)
```
stream qsv_stream {
  input rtsp://camera:554/stream;
  transcoder hw=qsv vb=2000k ab=128k;
}
```

### Device Selection
```
transcoder deviceid=0 hw=nvidia vb=2000k;
```

### GPU vs CPU
- GPU: Higher throughput, lower latency, multiple streams
- CPU: More flexible for complex transcoding
- Hybrid: Use GPU for encoding, CPU for other processing

## Output Profiles

### Profile Resolution Examples
```
# 4K
size=3840x2160

# Full HD 1080p
size=1920x1080

# HD 720p
size=1280x720

# SD 480p
size=640x480

# Mobile 360p
size=640x360

# Keep aspect ratio
size=1280x-2
```

### Bitrate Guidelines
- 4K: 4000-8000k
- 1080p: 2500-5000k
- 720p: 1000-2500k
- 480p: 500-1000k
- 360p: 300-600k
- Audio: 64-192k (AAC), 128-256k (MP3)

## Video Codecs

### Supported Output Codecs

**H.264 (AVC)**
- Most compatible
- Default codec
- Wide device support

**H.265 (HEVC)**
```
transcoder codec=h265 vb=2000k;
```
- Better compression (25-50% vs H.264)
- Limited browser support (Chrome 107+)
- Good for 4K/mobile

**AV1**
```
transcoder codec=av1 vb=2000k;
```
- Best compression (50% vs H.264)
- Limited hardware support
- Longer encoding time

### Audio Codecs
- AAC (default, recommended)
- MP3
- AC-3
- OPUS (for WebRTC)

**Select output audio:**
```
input rtsp://camera/stream output_audio=aac;
```

## Advanced Features

### Text Overlay (Watermark)
```
transcoder vb=2000k burn='{
  "text": {
    "text": "WATERMARK",
    "position": "br",
    "x": 10,
    "y": 10,
    "font_size": 20
  }
}';
```

**Positions:** tl (top-left), tr (top-right), bl (bottom-left), br (bottom-right), c (center)

**Text via API:**
```bash
curl -u admin:pass -X PUT http://localhost/api/v3/streams/demo \
  -H "Content-Type: application/json" \
  -d '{"transcoder": {"global": {"burn": {
    "text": {
      "position": "tr",
      "text": "Score: 3-2",
      "x": 10,
      "y": 10
    }
  }}}}'
```

### Logo Overlay
```
transcoder vb=2000k burn='{
  "image": {
    "path": "/path/to/logo.png",
    "position": "br",
    "x": 10,
    "y": 10
  }
}';
```

### Mosaic (Grid of Streams)
```
stream mosaic {
  input mosaic://cam1,cam2,cam3,cam4?fps=20&preset=ultrafast&bitrate=1024k&size=340x240&mosaic_size=16;
}
```

### Copy Streams (No Transcoding)
For formats that don't need transcoding:
```
stream copy_stream {
  input rtsp://source/stream;
  transcoder codec=copy ab=copy;
}
```
**Caution:** Ensure codecs are already compatible

## Flussonic Coder

### Hardware Solution
- NVIDIA Jetson-based hardware
- 6 Full HD or 12 SD streams per module
- Custom Linux OS with transcoding software
- No additional driver installation

### Configuration
```
stream coder_stream {
  input rtsp://camera:554/stream;
  transcoder hw=coder vb=2000k ab=128k deinterlace=yadif;
}
```

### Monitoring in UI
- Config > Chassis > Hardware Modules Monitor
- View: status, channels, temperature, power
- Module requirements: left-to-right slot insertion

### Module Capacity
- One NVIDIA Jetson: 6 x 1080p@30fps to 3 profiles
- Or: 12 x SD streams to 3 profiles
- Temperature monitoring critical

## Troubleshooting

### High CPU Usage
- Reduce number of profiles
- Use faster preset (ultrafast, fast)
- Enable hardware acceleration (GPU)
- Check stream FPS and resolution

### Transcoder Not Starting
- Check transcoder package installed: `apt list --installed | grep transcoder`
- Install if needed: `apt-get install flussonic-transcoder`
- For GPU: verify drivers installed
- Review logs: `/var/log/flussonic/flussonic.log`

### Sync Issues in Multibitrate
- Ensure all profiles have same GOP structure
- Use `fps=video` for all profiles
- Set same `preset` for all profiles
- Start all profiles from keyframes

### Quality Issues
- Use `preset=slow` for better quality
- Increase `vb` (bitrate)
- Check input source quality
- Verify codec compatibility

### Audio Missing
- Check input has audio
- Set `output_audio=aac` (or aac2)
- For WebRTC: use Opus
- Check ABR settings for audio

### GPU Not Used
```bash
# Check GPU visibility
nvidia-smi

# Verify process sees GPU
curl http://localhost/api/v3/system/info | jq '.gpu'
```

### Commands
```bash
# Check transcoder status
curl http://localhost/api/v3/streams/mystream | jq '.transcoder'

# View logs
tail -f /var/log/flussonic/flussonic.log

# Monitor GPU
nvidia-smi -l 1
watch -n 1 nvidia-smi
```
