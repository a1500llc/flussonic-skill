# Flussonic DVR Recording Guide

## Table of Contents
- [DVR Overview](#dvr-overview)
- [Configuration](#configuration)
- [Recording Streams](#recording-streams)
- [Storage](#storage)
- [Playback](#playback)
- [Archive Management](#archive-management)
- [Troubleshooting](#troubleshooting)

## DVR Overview

DVR (Digital Video Recording) allows recording live streams to disk for later playback. Features:
- Archive indefinite duration
- Time-shift playback
- Catch-up TV
- VOD from recordings
- Multi-profile support

## Configuration

### Basic DVR Setup
```
stream mystream {
  input rtsp://camera:554/stream;
  dvr;
}
```

### DVR with Custom Path
```
stream mystream {
  input rtsp://camera:554/stream;
  dvr /mnt/archive/mystream;
}
```

### DVR with Retention Policy
```
stream mystream {
  input rtsp://camera:554/stream;
  dvr /mnt/archive/mystream retention=7d;
}
```

**Retention Options:**
- `1d`, `7d`, `30d` - days
- `1h`, `24h` - hours
- Disk space used (storage limit takes precedence)

## Recording Streams

### Multiple Profiles
Record multiple bitrates for fallback:
```
stream hd {
  input rtsp://camera:554/stream;
  dvr /mnt/archive/hd;
  transcoder vb=5000k size=1920x1080 vb=2000k size=1280x720 vb=1000k size=640x480 ab=128k;
}
```

### Recording Parameters
- All profiles saved to same archive
- Separate segment files per profile
- Automatic index creation

### Status Check
```
# List active recordings
curl http://localhost/api/v3/streams | jq '.[] | select(.dvr) | {name, dvr}'

# View recording details
curl http://localhost/api/v3/streams/mystream | jq '.dvr'
```

## Storage

### Location Recommendations
- Dedicated fast storage (SSD or RAID)
- Separate mount point from OS
- Sufficient I/O for bitrate and concurrent streams

### Disk Requirements
```
Formula: bitrate (kbps) * seconds * streams
Example: 5000k * 86400s * 1 stream = 43.2 GB/day
         5000k * 86400s * 10 streams = 432 GB/day
```

### Multiple Disks
```
stream stream1 { dvr /mnt/disk1/stream1; }
stream stream2 { dvr /mnt/disk2/stream2; }
stream stream3 { dvr /mnt/disk1/stream3; }
```

### Monitoring Storage
```bash
# Check disk usage
df -h /mnt/archive

# Archive size
du -sh /mnt/archive/stream*

# Watch real-time
watch -n 5 'du -sh /mnt/archive/*'
```

## Playback

### Archive Playback URL
```
# HLS (recommended)
http://FLUSSONIC-IP/mystream/archive-YYYYMMDD/HHMMSS.m3u8

# Example for 2024-02-15 14:30:00
http://FLUSSONIC-IP/mystream/archive-20240215/143000.m3u8

# MPEG-TS
http://FLUSSONIC-IP/mystream/archive-YYYYMMDD/HHMMSS.ts
```

### Time-Shift Window
View live stream with rewind capability:
```
http://FLUSSONIC-IP/mystream/timeshift.m3u8
```

### Embed in HTML
```html
<video controls width="640" height="480">
  <source src="http://server/stream/archive-20240215/143000.m3u8"
    type="application/vnd.apple.mpegurl">
</video>
```

### API Archive Query
```bash
# List available segments
curl http://localhost/api/v3/streams/mystream/segments

# Get specific date range
curl "http://localhost/api/v3/streams/mystream/segments?from=2024-02-15T14:00:00Z&to=2024-02-15T16:00:00Z"
```

## Archive Management

### Cleanup Old Archives
Manual deletion:
```bash
# Delete older than 30 days
find /mnt/archive -type f -mtime +30 -delete
```

### Export Segment
```bash
# Download archive segment
curl -o segment.ts \
  "http://localhost/api/v3/streams/mystream/segments/20240215-143000.ts"
```

### Convert Archive to VOD
1. Export segment to VOD location
2. Rename with .mp4 extension
3. Access via VOD interface

### Disk Space Cleanup
```bash
# Immediate retention enforcement
service flussonic reload

# Monitor archive size
du -sh /mnt/archive
df -h /mnt/archive
```

## Troubleshooting

### Recording Not Starting
- Verify DVR directory exists and is writable
- Check disk space: `df -h`
- Review logs: `tail -f /var/log/flussonic/flussonic.log`
- Check stream is active: UI > Media > Streams

### High Disk Usage
- Check retention policy being enforced
- Monitor bitrate: API > stream > stats > bitrate
- Review number of profiles being recorded
- Check for stale processes: `ps aux | grep flussonic`

### Playback Issues
- Verify archive segments exist: `ls -la /mnt/archive/stream/`
- Check segment timestamps match URL
- Ensure stream was recording at requested time
- Test with HTTP directly (not HTTPS if certificate issues)

### Corrupted Segments
- Check I/O errors: `dmesg | tail -20`
- Monitor disk health: `smartctl -a /dev/sda`
- Restart recording: reload stream config

### Commands
```bash
# Check DVR status
curl http://localhost/api/v3/streams/mystream | grep -A 5 dvr

# View recent archives
ls -lht /mnt/archive/mystream/ | head

# Check for errors
grep -i "dvr\|record" /var/log/flussonic/flussonic.log
```
