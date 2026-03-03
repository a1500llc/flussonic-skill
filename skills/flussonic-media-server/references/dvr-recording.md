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

DVR requires two steps: (1) define a global `dvr` block with storage settings, then (2) reference it in streams with `@name`.

### Step 1: Global DVR Config
```
dvr my_storage {
  root /mnt/archive;
}
```

For RAID setups with multiple disks:
```
dvr my_raid {
  root /storage/raid;
  raid 0;
  metadata idx;
  disk volume1;
  disk volume2;
  disk volume3;
}
```

### Step 2: Reference DVR in Streams
```
stream mystream {
  input rtsp://camera:554/stream;
  dvr @my_storage 7d;
}
```

### Retention & Limits
Retention and disk limits go after the `@name` reference:
```
dvr @my_storage 7d;           # 7 days retention
dvr @my_storage 2d 97%;       # 2 days, max 97% disk usage
dvr @my_storage 90% 5d;       # also accepts inverted order
dvr @my_storage 3d 500G;      # 3 days, max 500GB
dvr @my_storage;               # no retention limit (records indefinitely)
```

**Retention Options:**
- `1d`, `7d`, `30d` - days
- `1h`, `24h` - hours
- `90%`, `97%` - max disk usage percentage
- `500G`, `10G` - max storage size

### DVR with Replication
```
dvr @my_storage 7d replicate;
dvr @my_storage 7d replicate replication_port=8002;
```

### Cloud DVR (S3)
```
dvr cloud_storage {
  root s3://accesskey:secretkey@endpoint/bucket;
}

stream mystream {
  input udp://239.0.0.1:1234;
  dvr @cloud_storage 10G;
}
```

### Copy to secondary location
```
dvr @my_storage 7d;
# Also supported: copy= syntax for backup
dvr /storage copy=s3://key:secret@endpoint/bucket 10G;
```

## Recording Streams

### Multiple Profiles with DVR
Record multiple bitrates:
```
dvr archive1 {
  root /mnt/archive;
}

stream hd {
  input rtsp://camera:554/stream;
  dvr @archive1 7d;
  transcoder vb=5000k size=1920x1080 vb=2000k size=1280x720 vb=1000k size=640x480 ab=128k;
}
```

### Recording Parameters
- All profiles saved to same archive
- Separate segment files per profile
- Automatic index creation

### Status Check
```bash
# List DVR configurations
curl -u user:pass http://server/streamer/api/v3/dvrs

# View stream DVR ranges
curl -u user:pass http://server/streamer/api/v3/streams/mystream/dvr/ranges
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

### Multiple Disks (Flussonic RAID)
Use Flussonic's application-level RAID to manage multiple disks automatically:
```
dvr my_raid {
  root /storage;
  raid 0;
  metadata idx;
  disk d1;
  disk d2;
  disk d3;
  disk d4;
  active 2;  # write to 2 disks simultaneously (saves power)
}

stream stream1 { input ...; dvr @my_raid 7d; }
stream stream2 { input ...; dvr @my_raid 7d; }
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
# List available DVR ranges
curl -u user:pass http://server/streamer/api/v3/streams/mystream/dvr/ranges

# Export DVR segment as MP4
curl -u user:pass -X POST http://server/streamer/api/v3/streams/mystream/dvr/export

# Check DVR consistency
curl -u user:pass -X POST http://server/streamer/api/v3/streams/mystream/dvr/consistency_check
```

Note: The endpoint `/api/v3/streams/{name}/segments` does NOT exist. Use `/streamer/api/v3/streams/{name}/dvr/ranges` instead.

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
  "http://server/mystream/archive-20240215-3600.ts"
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
curl -u user:pass http://server/streamer/api/v3/streams/mystream | grep -A 5 dvr

# View recent archives
ls -lht /mnt/archive/mystream/ | head

# Check for errors
grep -i "dvr\|record" /var/log/flussonic/flussonic.log
```
