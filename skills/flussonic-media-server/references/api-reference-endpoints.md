# Flussonic API Reference - Control & Configuration Endpoints

## Table of Contents
- [Overview](#overview)
- [Streams Management](#streams-management)
- [Configuration & Stats](#configuration--stats)
- [VOD Management](#vod-management)
- [Monitoring & Health](#monitoring--health)
- [DVR & Archive](#dvr--archive)
- [DVR Configurations (Global)](#dvr-configurations-global)
- [Templates](#templates)
- [Authentication & Sessions](#authentication--sessions)
- [Cluster](#cluster)
- [Common Patterns](#common-patterns)

## Overview

**Base URL:** `http://FLUSSONIC-IP/streamer/api/v3/`
**Authentication:** Basic auth using `edit_auth` credentials (for write) or `view_auth` (for read-only)
**Response Format:** JSON
**API Design:** RESTful with UPSERT pattern (PUT creates or updates)

### Authentication Methods
```bash
# Basic Auth (edit_auth credentials from flussonic.conf)
curl -u username:password http://server/streamer/api/v3/streams

# Bearer Token
curl -H "Authorization: Bearer TOKEN" http://server/streamer/api/v3/streams
```

### General Patterns

**Create/Update (Upsert) — always PUT, not POST:**
```bash
PUT /streamer/api/v3/streams/{name}
```

**Delete:**
```bash
DELETE /streamer/api/v3/streams/{name}
```

**List (returns collection with metadata):**
```bash
GET /streamer/api/v3/streams
```

**Get Single:**
```bash
GET /streamer/api/v3/streams/{name}
```

### Collection Response Format
All list endpoints return this structure:
```json
{
  "streams": [...],
  "estimated_count": 100,
  "timing": {"select": 0, "sort": 0, "load": 5, "filter": 0, "limit": 0},
  "next": "cursor_value",
  "prev": "cursor_value"
}
```

### Pagination (Cursor-Based)
Flussonic uses **cursor-based pagination**, NOT offset-based:
```bash
GET /streamer/api/v3/streams?limit=50
GET /streamer/api/v3/streams?limit=50&cursor=CURSOR_VALUE
```

### Filtering & Sorting
```bash
# Filter by field value
GET /streamer/api/v3/streams?stats.status=running

# Comparison operators: _lt, _lte, _gt, _gte, _like, _is=null
GET /streamer/api/v3/streams?stats.bitrate_gt=1000

# Sort (prefix with - for descending)
GET /streamer/api/v3/streams?sort=name,-stats.bitrate

# Select specific fields
GET /streamer/api/v3/streams?select=name,stats.status,stats.bitrate
```

## Streams Management

### List Streams
```bash
GET /streamer/api/v3/streams
```
Returns all streams with config, stats, inputs, transcoder info.

### Get Stream Details
```bash
GET /streamer/api/v3/streams/{name}
```

### Create/Update Stream (Upsert via PUT)
```bash
PUT /streamer/api/v3/streams/{name}
Content-Type: application/json
```

**Correct JSON body format:**
```json
{
  "inputs": [
    {"url": "udp://239.0.0.1:1234"}
  ],
  "transcoder": {
    "global": {
      "hw": "nvenc",
      "deviceid": 0,
      "fps": 30,
      "gop": 60
    },
    "video": [
      {"vb": "2000k", "size": "1280x720"},
      {"vb": "800k", "size": "640x360"}
    ],
    "audio": {
      "ab": "128k",
      "acodec": "aac"
    }
  },
  "dvr": {
    "reference": "my_storage",
    "expiration": 604800
  },
  "protocols": {
    "hls": true,
    "dash": true
  },
  "segment_duration": 2000,
  "segment_count": 30
}
```

**Important notes:**
- `inputs` is an **array of objects** with `"url"` field, NOT a string
- `transcoder` is a structured object, NOT flat `{"vb": "2000k"}`
- `dvr` is an object (with `reference` to global DVR, or `root` for inline), NOT a string path
- Partial updates: send only changed fields (merges with existing config)
- Full replace: add `"$reset": true` to replace entire config
- Set a field to `null` to disable a feature: `{"drm": null}`

### Delete Stream
```bash
DELETE /streamer/api/v3/streams/{name}
```
Returns 204 No Content on success.

### Stop Stream
```bash
POST /streamer/api/v3/streams/{name}/stop
```
Stops processing without deleting config.

### Force Select Input
```bash
POST /streamer/api/v3/streams/{name}/inputs/{index}/select
```
Force switch to a specific input source (private API).

### Stream State Values
- `running` - Stream active and processing
- `waiting` - Waiting for source to connect
- `starting` - Initializing connection
- `stopped` - Manually stopped

## Configuration & Stats

### Get Server Config + Stats
```bash
GET /streamer/api/v3/config
```
Returns complete server configuration AND live stats including:
- Server version, uptime, hostname
- CPU/memory/scheduler usage
- Transcoder devices (GPUs)
- Disk partitions and usage
- Listener configuration (HTTP/HTTPS ports)
- edit_auth credentials
- Server names

### Get Stats Only
```bash
GET /streamer/api/v3/config/stats
```
Returns runtime statistics: CPU, memory, disk, GPU, streams count, network throughput.

### Update Server Config
```bash
PUT /streamer/api/v3/config
```

**Note:** The following endpoints do NOT exist:
- ~~`GET /api/v3/config/settings`~~ — use `GET /streamer/api/v3/config`
- ~~`GET /api/v3/config/listeners`~~ — listener info is in config response
- ~~`GET /api/v3/system/info`~~ — use `GET /streamer/api/v3/config/stats`

## VOD Management

### List VOD Locations
```bash
GET /streamer/api/v3/vods
```

### Get VOD Details
```bash
GET /streamer/api/v3/vods/{prefix}
```

### Create/Update VOD Location
```bash
PUT /streamer/api/v3/vods/{prefix}
Content-Type: application/json

{
  "storages": [{"path": "/storage/videos"}]
}
```

### Delete VOD Location
```bash
DELETE /streamer/api/v3/vods/{prefix}
```

### List Files in VOD
```bash
GET /streamer/api/v3/vods/{prefix}/storages/{storage_index}/files
GET /streamer/api/v3/vods/{prefix}/storages/{storage_index}/files/{subpath}
```

### Get Opened VOD Files
```bash
GET /streamer/api/v3/vods/opened_files
```

## Monitoring & Health

### Liveness Check
```bash
GET /streamer/api/v3/monitoring/liveness
```
Returns server version and current timestamp. Use for basic health checks.

**Response:**
```json
{
  "now": 1772544650689,
  "build": 0,
  "server_version": "26.02"
}
```

### Readiness Check
```bash
GET /streamer/api/v3/monitoring/readiness
```
Returns version, start time, and current timestamp. Use for load balancer health checks.

**Response:**
```json
{
  "now": 1772544650701,
  "build": 0,
  "started_at": 1771531360,
  "server_version": "26.02"
}
```

**Note:** The following endpoints do NOT exist:
- ~~`GET /api/v3/health`~~
- ~~`GET /api/v3/stats`~~
- ~~`GET /api/v3/system/info`~~

## DVR & Archive

### Get DVR Ranges
```bash
GET /streamer/api/v3/streams/{name}/dvr/ranges
```
Returns available recording time windows.

### Delete DVR Range
```bash
DELETE /streamer/api/v3/streams/{name}/dvr/ranges
Query: from=TIMESTAMP&to=TIMESTAMP
```

### Export DVR as MP4
```bash
POST /streamer/api/v3/streams/{name}/dvr/export
```

### Check DVR Consistency
```bash
POST /streamer/api/v3/streams/{name}/dvr/consistency_check
```

### DVR Locks (Deprecated)
```bash
GET /streamer/api/v3/streams/{name}/dvr/locks     # List locks
POST /streamer/api/v3/streams/{name}/dvr/locks    # Create lock
DELETE /streamer/api/v3/streams/{name}/dvr/locks  # Remove lock
```
Note: DVR locks endpoints are **deprecated** in current versions.

### DVR Export Jobs
```bash
GET /streamer/api/v3/dvr_export_jobs              # List background export jobs
PUT /streamer/api/v3/dvr_export_jobs/{id}         # Start a job
GET /streamer/api/v3/dvr_export_jobs/{id}         # Get job status
DELETE /streamer/api/v3/dvr_export_jobs/{id}      # Cancel a job
```

**Note:** The endpoint ~~`GET /api/v3/streams/{name}/segments`~~ does NOT exist. Use `GET /streamer/api/v3/streams/{name}/dvr/ranges` instead.

## DVR Configurations (Global)

### List DVR Configs
```bash
GET /streamer/api/v3/dvrs
```

### Get/Create/Update/Delete DVR Config
```bash
GET /streamer/api/v3/dvrs/{name}
PUT /streamer/api/v3/dvrs/{name}
DELETE /streamer/api/v3/dvrs/{name}
```

**Create DVR config:**
```json
{
  "root": "/mnt/archive",
  "expiration": 604800,
  "disk_usage_limit": 97,
  "storage_limit": 400000000000,
  "dvr_replicate": false,
  "replication_port": 8002
}
```

### DVR Disk Management
```bash
GET /streamer/api/v3/dvrs/{name}/disks
GET /streamer/api/v3/dvrs/{name}/disks/{path}
PUT /streamer/api/v3/dvrs/{name}/disks/{path}
DELETE /streamer/api/v3/dvrs/{name}/disks/{path}
```

## Templates

### List Templates
```bash
GET /streamer/api/v3/templates
```

### Get/Create/Update/Delete Template
```bash
GET /streamer/api/v3/templates/{name}
PUT /streamer/api/v3/templates/{name}
DELETE /streamer/api/v3/templates/{name}
```

**Example template (for accepting RTMP publish):**
```json
{
  "inputs": [{"url": "publish://"}],
  "protocols": {"hls": true, "dash": true}
}
```

## Authentication & Sessions

### Auth Backends
```bash
GET /streamer/api/v3/auth_backends
GET /streamer/api/v3/auth_backends/{name}
PUT /streamer/api/v3/auth_backends/{name}
DELETE /streamer/api/v3/auth_backends/{name}
```

### Sessions
```bash
GET /streamer/api/v3/sessions                # List active sessions
GET /streamer/api/v3/sessions/{id}           # Get session details
DELETE /streamer/api/v3/sessions/{id}        # Kill session
POST /streamer/api/v3/sessions/reauth        # Re-authenticate sessions
```

### API Tokens
```bash
GET /streamer/api/v3/api_tokens              # List API tokens (read-only)
```

**Note:** The following token management endpoints do NOT exist:
- ~~`POST /api/v3/tokens`~~ (create token)
- ~~`DELETE /api/v3/tokens/{id}`~~ (revoke token)
- ~~`PUT /api/v3/tokens/{id}`~~ (refresh token)

Token-based stream authentication is configured via auth backends (on_play/on_publish callbacks) or Lua scripts, not through a token CRUD API.

### Event Sinks
```bash
GET /streamer/api/v3/event_sinks
GET /streamer/api/v3/event_sinks/{name}
PUT /streamer/api/v3/event_sinks/{name}
DELETE /streamer/api/v3/event_sinks/{name}
GET /streamer/api/v3/event_sinks/{name}/events
```

## Cluster

### Peers
```bash
GET /streamer/api/v3/cluster/peers
GET /streamer/api/v3/cluster/peers/{hostname}
PUT /streamer/api/v3/cluster/peers/{hostname}
DELETE /streamer/api/v3/cluster/peers/{hostname}
```

### Sources
```bash
GET /streamer/api/v3/cluster/sources
GET /streamer/api/v3/cluster/sources/{url}
PUT /streamer/api/v3/cluster/sources/{url}
DELETE /streamer/api/v3/cluster/sources/{url}
```

## Common Patterns

### Error Response
```json
{
  "error": "not_found",
  "name": "stream_name"
}
```

### Stream Monitoring Script
```bash
# List all streams with status and bitrate
curl -s -u user:pass http://server/streamer/api/v3/streams | \
  python3 -c 'import sys,json; data=json.load(sys.stdin);
[print(f"{s[\"name\"]}: {s.get(\"stats\",{}).get(\"status\",\"?\")} ({s.get(\"stats\",{}).get(\"bitrate\",0)} kbps)") for s in data.get("streams",[])]'
```

### Create Stream via API
```bash
curl -u user:pass -X PUT \
  http://server/streamer/api/v3/streams/test_stream \
  -H "Content-Type: application/json" \
  -d '{"inputs": [{"url": "fake://fake"}]}'
```

### Get Server Stats
```bash
curl -s -u user:pass http://server/streamer/api/v3/config/stats | python3 -m json.tool
```

### Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success (GET, PUT) |
| 204 | No Content (DELETE success) |
| 400 | Bad Request |
| 401 | Unauthorized |
| 404 | Not Found |
| 500 | Server Error |

### OpenAPI Schema
The complete API schema is available at:
```bash
GET /streamer/api/v3/schema
```
Returns OpenAPI 3.0/3.1 specification for all endpoints.

## Additional Endpoints

### Logos
```bash
GET/PUT/DELETE /streamer/api/v3/logos/{name}
```

### Caches
```bash
GET/PUT/DELETE /streamer/api/v3/caches/{name}
```

### IPTV
```bash
GET/PUT/DELETE /streamer/api/v3/iptv
```

### DVB Cards
```bash
GET/PUT/DELETE /streamer/api/v3/dvb_cards/{name}
GET /streamer/api/v3/dvb_cards/{name}/available_programs
```

### Episodes
```bash
GET /streamer/api/v3/episodes
GET /streamer/api/v3/dvr_episodes
```
