# Flussonic API Reference - Control & Configuration Endpoints

## Table of Contents
- [Overview](#overview)
- [Streams Management](#streams-management)
- [Configuration](#configuration)
- [VOD Management](#vod-management)
- [System & Status](#system--status)
- [DVR & Archive](#dvr--archive)
- [Templates](#templates)
- [Authentication & Tokens](#authentication--tokens)
- [Common Patterns](#common-patterns)

## Overview

**Base URL:** `http://localhost/api/v3/`
**Authentication:** Basic auth (username:password)
**Response Format:** JSON
**Version:** 3 (current stable)

### General Patterns

**Create/Update (Upsert):**
```bash
PUT /api/v3/resource/{name}
```

**Delete:**
```bash
DELETE /api/v3/resource/{name}
```

**List:**
```bash
GET /api/v3/resources
```

**Get Single:**
```bash
GET /api/v3/resource/{name}
```

## Streams Management

### List Streams
```bash
GET /api/v3/streams
```
Returns all streams with current state, stats, input/output info

### Get Stream Details
```bash
GET /api/v3/streams/{name}
```
Full configuration and detailed stats for specific stream

### Create/Update Stream
```bash
PUT /api/v3/streams/{name}
Content-Type: application/json

{
  "input": "rtsp://camera:554/stream",
  "hls": {},
  "dvr": "/archive/mystream",
  "transcoder": {
    "vb": "2000k",
    "ab": "128k",
    "size": "1280x720"
  }
}
```

### Delete Stream
```bash
DELETE /api/v3/streams/{name}
```

### Stop Stream
```bash
POST /api/v3/streams/{name}/stop
```
Stops processing without deleting config

### Stream State Values
- `running` - Stream active and processing
- `starting` - Initializing
- `disconnected` - Input disconnected
- `stalled` - Error state
- `disabled` - Not configured

## Configuration

### Get Server Config
```bash
GET /api/v3/config
```
Returns complete server configuration (flussonic.conf equivalent)

### Save Server Config
```bash
PUT /api/v3/config
Content-Type: application/json

{
  "edit_auth": "admin password",
  "listeners": [
    { "address": "0.0.0.0", "port": 80 }
  ]
}
```

### Server Settings
```bash
GET /api/v3/config/settings
PUT /api/v3/config/settings
```
Modify: HTTP ports, HTTPS, admin credentials, license key

### Listeners
```bash
GET /api/v3/config/listeners
POST /api/v3/config/listeners
```
Manage HTTP/HTTPS ports, SSL certificates, API access

## VOD Management

### List VOD Locations
```bash
GET /api/v3/vods
```

### Get VOD Details
```bash
GET /api/v3/vods/{name}
```
Lists files in VOD location

### Create VOD Location
```bash
PUT /api/v3/vods/{name}
Content-Type: application/json

{
  "location": "/storage/videos"
}
```

### Delete VOD Location
```bash
DELETE /api/v3/vods/{name}
```

### Upload File to VOD
```bash
POST /api/v3/vods/{name}/upload
Content-Type: multipart/form-data

file: <binary file>
```

### List Files in VOD
```bash
GET /api/v3/vods/{name}/files
GET /api/v3/vods/{name}/files/{path}
```

## System & Status

### System Info
```bash
GET /api/v3/system/info
```
Returns: CPU, memory, disk, GPU, license, version

### Stream Statistics
```bash
GET /api/v3/stats
```
Real-time stats for all streams (bitrate, viewers, uptime)

### Stream-Specific Stats
```bash
GET /api/v3/streams/{name}/stats
```

### Server Status
```bash
GET /api/v3/health
```
Health check endpoint (returns 200 if running)

## DVR & Archive

### Get DVR Ranges
```bash
GET /api/v3/streams/{name}/dvr/ranges
```
Returns available recording time windows

### Delete DVR Range
```bash
DELETE /api/v3/streams/{name}/dvr/ranges
Query: from=TIMESTAMP&to=TIMESTAMP
```

### Lock DVR Segment
```bash
POST /api/v3/streams/{name}/dvr/locks
{
  "from": "2024-02-25T10:00:00Z",
  "to": "2024-02-25T12:00:00Z"
}
```
Prevent deletion of segment

### List DVR Locks
```bash
GET /api/v3/streams/{name}/dvr/locks
```

### Get DVR Segments
```bash
GET /api/v3/streams/{name}/segments
Query: from=DATE&to=DATE
```

## Templates

### List Templates
```bash
GET /api/v3/templates
```

### Get Template
```bash
GET /api/v3/templates/{name}
```

### Create Template
```bash
PUT /api/v3/templates/{name}
{
  "input": "publish://",
  "hls": {}
}
```

### Delete Template
```bash
DELETE /api/v3/templates/{name}
```

## Authentication & Tokens

### List Tokens
```bash
GET /api/v3/tokens
```

### Create Token
```bash
POST /api/v3/tokens
{
  "name": "viewer1",
  "streams": ["camera1", "camera2"],
  "expires": 3600
}
```

### Revoke Token
```bash
DELETE /api/v3/tokens/{token_id}
```

### Refresh Token
```bash
PUT /api/v3/tokens/{token_id}
{
  "expires": 7200
}
```

### Token Response
```json
{
  "token": "abc123def456...",
  "name": "viewer1",
  "streams": ["camera1", "camera2"],
  "created": "2024-02-25T10:00:00Z",
  "expires": "2024-02-25T11:00:00Z"
}
```

## Common Patterns

### Error Response
```json
{
  "error": "stream_not_found",
  "message": "Stream 'unknown' not found"
}
```

### Batch Operations
```bash
# Get multiple streams at once
curl http://localhost/api/v3/streams | jq '.[] | select(.name | test("camera"))'

# Filter active streams
curl http://localhost/api/v3/streams | jq '.[] | select(.state=="running")'
```

### Monitoring Script
```bash
# Check all stream statuses
for stream in $(curl -s http://localhost/api/v3/streams | jq -r '.[].name'); do
  state=$(curl -s http://localhost/api/v3/streams/$stream | jq -r '.state')
  bitrate=$(curl -s http://localhost/api/v3/streams/$stream | jq -r '.stats.bitrate')
  echo "$stream: $state ($bitrate kbps)"
done
```

### Common Query Parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `token` | Stream access token | `?token=abc123` |
| `from` | Start time/date | `?from=2024-02-25T10:00:00Z` |
| `to` | End time/date | `?to=2024-02-25T12:00:00Z` |
| `limit` | Result limit | `?limit=100` |
| `offset` | Pagination offset | `?offset=50` |
| `filter` | Search/filter | `?filter=camera` |

### Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 404 | Not Found |
| 409 | Conflict |
| 500 | Server Error |

## Useful Combinations

### Restart Stream
```bash
curl -X POST http://localhost/api/v3/streams/mystream/stop
sleep 2
curl http://localhost/api/v3/streams | jq '.[] | select(.name=="mystream")'
```

### Export Config
```bash
curl -u admin:pass http://localhost/api/v3/config > flussonic.json
```

### Monitor Stream Health
```bash
watch -n 5 'curl -s http://localhost/api/v3/stats | jq ".streams[] | {name, bitrate, viewers}"'
```

### Get Stream with Most Viewers
```bash
curl -s http://localhost/api/v3/stats | \
  jq '.streams | max_by(.viewers)'
```
