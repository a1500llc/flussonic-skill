# Flussonic VOD (Video On Demand) Guide

## Table of Contents
- [VOD Overview](#vod-overview)
- [Configuration](#configuration)
- [File Organization](#file-organization)
- [Supported Formats](#supported-formats)
- [Playback URLs](#playback-urls)
- [File Management](#file-management)
- [Streaming VOD Files](#streaming-vod-files)
- [Troubleshooting](#troubleshooting)

## VOD Overview

VOD allows playing pre-recorded video files on demand. Features:
- File-based playback
- Directory-based organization
- Multiple protocol support
- Index generation
- Playlist support

## Configuration

### Basic VOD Location
```
vod movies {
  location = /storage/videos;
}
```

### Multiple VOD Locations
```
vod movies {
  location = /storage/movies;
}

vod tv_shows {
  location = /storage/shows;
}

vod archives {
  location = /dvr/archives;
}
```

### VOD with Prefix Mapping
URL prefix maps to directory:
```
# URL: http://server/movies/bigbuckbunny.mp4
# Maps to: /storage/videos/bigbuckbunny.mp4
vod movies {
  location = /storage/videos;
}
```

## File Organization

### Directory Structure Example
```
/storage/
  videos/
    movie1.mp4
    movie2.mp4
    show1/
      episode1.mp4
      episode2.mp4
    show2/
      episode1.mp4
```

### Naming Conventions
- Use descriptive names
- Avoid special characters
- Use lowercase extensions (.mp4, .mov, .m4v)
- Group by series in subdirectories

### File Permissions
```bash
# Ensure readable by flussonic user
chmod -R 755 /storage/videos
chown -R flussonic:flussonic /storage/videos
```

## Supported Formats

### Container Formats
- `.mp4` - H.264/AAC (recommended)
- `.f4v` - Flash video (legacy)
- `.mov` - QuickTime
- `.m4v` - iTunes video
- `.mp4a` - Audio
- `.3gp`, `.3g2` - Mobile video

### Video Codecs
- **H.264 (AVC)** - Primary
- **H.265 (HEVC)** - Supported in latest browsers
- Note: MPEG-2 requires transcoding

### Audio Codecs
- **AAC** - Primary (all profiles)
- **MP3** - Supported

### File Requirements
- Container and codec combination supported
- Proper file encoding
- Readable by ffmpeg

## Playback URLs

### Direct File Playback
```
http://FLUSSONIC-IP/vod_name/path/file.mp4
```

### Examples
```
# Single file
http://server/movies/BigBuckBunny.mp4

# In subdirectory
http://server/shows/Season1/Episode1.mp4
```

### Protocol Support
```
# HTTP
http://server/movies/file.mp4

# HTTPS
https://server/movies/file.mp4

# HLS adaptive
http://server/movies/file.mp4/index.m3u8

# DASH
http://server/movies/file.mp4/manifest.mpd

# HTTP Smooth Streaming
http://server/movies/file.mp4/Manifest
```

### Streaming Formats
VOD files automatically available in:
- Raw MP4
- HLS with adaptive bitrates
- DASH with segmentation
- HTTP Progressive Download

## File Management

### Upload Files via UI
1. Media > VODs > [VOD name]
2. Click **Browse** > **Upload Files**
3. Select file from computer
4. Wait for upload to complete

### Upload via Command Line
```bash
# Copy file to VOD directory
cp /local/path/video.mp4 /storage/videos/

# Or
scp video.mp4 user@server:/storage/videos/
```

### Delete Files
```bash
# Via UI: Browse > Select file > Delete

# Via command line
rm /storage/videos/video.mp4
```

### List Files
```bash
# Via UI: Media > VODs > [VOD name] > Browse

# Via command line
ls -lh /storage/videos/
find /storage/videos -name "*.mp4"
```

## Streaming VOD Files

### Create VOD Stream from File
```
stream vod_movie {
  input file:///storage/videos/BigBuckBunny.mp4;
}
```

### Playlist (Multiple Files)
```
stream playlist {
  input file:///storage/videos/file1.mp4;
  input file:///storage/videos/file2.mp4;
  input file:///storage/videos/file3.mp4;
}
```

### Loop Configuration
```
stream looped {
  input file:///storage/videos/promo.mp4 loop;
}
```

### File Sequence
```
stream sequence {
  input file:///storage/videos/intro.mp4;
  input file:///storage/videos/content.mp4;
  input file:///storage/videos/outro.mp4;
}
```

## Troubleshooting

### File Not Accessible
```bash
# Check file exists
ls -la /storage/videos/filename.mp4

# Check permissions
stat /storage/videos/filename.mp4

# Verify readable by flussonic
su - flussonic -s /bin/bash
head -c 100 /storage/videos/filename.mp4
```

### Playback Issues
- **Invalid format:** Check codec support (H.264 + AAC required)
- **No video:** Verify file is not corrupted: `ffprobe filename.mp4`
- **No audio:** Check audio track exists and is readable
- **Stalls:** Check disk I/O, network bandwidth

### Unsupported Codec
```bash
# Check file codec
ffprobe -v error -select_streams v:0 -show_entries \
  stream=codec_name -of default=noprint_wrappers=1:nokey=1 video.mp4

# If MPEG-2 or unsupported, transcode
ffmpeg -i input.mpeg2 -c:v libx264 -c:a aac output.mp4
```

### Disk Space
```bash
# Check VOD directory size
du -sh /storage/videos/

# Find large files
find /storage/videos -type f -size +1G

# Free space
df -h /storage/
```

### Commands
```bash
# Check VOD configuration
curl http://localhost/api/v3/vods | jq

# List files
curl http://localhost/api/v3/vods/movies | jq

# Check file details
ffprobe http://localhost/movies/file.mp4 -show_format
```

### Performance Tips
- Use SSD storage for fast seeking
- Keep files under 4GB for compatibility
- Index files on first play (creates .index file)
- Monitor disk I/O during playback
- Use local storage, not network shares

