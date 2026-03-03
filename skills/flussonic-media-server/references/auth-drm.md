# Flussonic Authentication & DRM Guide

## Table of Contents
- [Authentication Types](#authentication-types)
- [Access Control](#access-control)
- [Token-Based Auth](#token-based-auth)
- [Secure Link](#secure-link)
- [DRM Systems](#drm-systems)
- [CORS Configuration](#cors-configuration)
- [Session Management](#session-management)
- [Troubleshooting](#troubleshooting)

## Authentication Types

### Admin UI Authentication
```
edit_auth admin password;        # Full admin access
view_auth viewer readpass;       # Read-only API
```

### Stream Access Authentication
```
stream protected {
  input rtsp://camera:554/stream;
  auth token;                    # Token-based auth
}
```

## Access Control

### IP-Based Whitelist
```
http 80 {
  address 192.168.1.0/24;        # Allow only this subnet
}

https 443 {
  address 10.0.0.100;            # Single IP only
}
```

### Port-Based API Control
Disable API on specific ports:
```
https 443 {
  api false;                     # API disabled on HTTPS
}

http 8080 {
  api true;                      # API enabled on 8080
}
```

### Max Sessions Per User
```
stream protected {
  input rtsp://camera:554/stream;
  auth token;
  max_sessions = 3;              # Max 3 concurrent viewers
}
```

### IP-Based Bans
```
stream protected {
  input rtsp://camera:554/stream;
  auth token;
  ban_per_ip;                    # Ban IPs with invalid tokens
}
```

## Token-Based Auth

Flussonic implements token-based authentication via **auth backends** (on_play/on_publish callbacks) or **Lua scripts**, NOT through a token CRUD API.

### Auth Backend (Middleware Callback)
Configure stream to call an external auth server:
```
stream protected_stream {
  input rtsp://camera:554/stream;
  on_play http://auth-server/verify;
}
```

When a viewer requests playback, Flussonic calls the auth backend with stream name, token, IP, etc. The auth server responds with 200 (allow) or 403 (deny).

### Playback with Token
```
http://server/camera1/index.m3u8?token=abc123def456
http://server/camera1/index.ts?token=abc123def456
```

### Lua-Based Token Auth
Flussonic supports Lua scripts for custom token validation:
```
# In flussonic.conf, reference a Lua script
# Lua scripts stored in /etc/flussonic/ or /opt/flussonic/lua/
```

### Auth Backend API
```bash
# Manage auth backends
GET /streamer/api/v3/auth_backends
PUT /streamer/api/v3/auth_backends/{name}
DELETE /streamer/api/v3/auth_backends/{name}

# View sessions
GET /streamer/api/v3/sessions
DELETE /streamer/api/v3/sessions/{id}
```

Note: The endpoints `/api/v3/tokens`, `POST /api/v3/tokens`, `DELETE /api/v3/tokens/{id}` do NOT exist. Token management is done through auth backends and Lua scripts.

## Secure Link

Secure links use Lua-based token generation (securetoken.lua) with time-limited URLs and IP binding:
```
http://server/camera1/index.m3u8?token=GENERATED_TOKEN
```

Token validation is done via Lua scripts or auth backends. See `references/server-internals.md` for Lua scripting API details.

## DRM Systems

Flussonic uses the **CPIX standard** for multi-DRM protection. Instead of configuring each DRM system separately, you configure a CPIX keyserver that handles Widevine, PlayReady, and FairPlay simultaneously.

### CPIX DRM (Recommended — Handles All DRM Systems)
```
stream protected_channel {
  input udp://239.0.0.1:1234;
  drm cpix keyserver=http://drm-server/api/drm/cpix?client=myapp&clientToken=SECRET resource_id=protected_channel;
  protocols dash hls;
}
```

This single directive enables:
- **Widevine** — for Chrome, Android, smart TVs
- **PlayReady** — for Edge, Windows, Xbox
- **FairPlay** — for Safari, iOS, Apple TV

The CPIX keyserver manages key rotation, content keys, and license delivery for all DRM systems.

### ClearKey DRM (for DASH, since v25.09)
```
stream clearkey_stream {
  input udp://239.0.0.1:1234;
  drm clearkey;
  protocols dash;
}
```

### Real-World DRM Config Example
From production Flussonic servers:
```
stream Live_1196 {
  input udp://239.77.16.88:8999/172.17.202.18?sources=10.77.22.88&buffer_size=8M;
  drm cpix keyserver=http://localhost:8888/api/drm/cpix?client=dtvgo&clientToken=flussonic-TOKEN resource_id=Live_1196;
  protocols dash hls;
}
```

Note: The syntax `drm widevine { license_url = "..."; }` with separate blocks per DRM vendor is NOT how modern Flussonic configures DRM. Use `drm cpix keyserver=URL resource_id=NAME;` instead.

## CORS Configuration

### Enable CORS
```
http 80 {
  cors_enabled = yes;
  cors_origins = "https://player.example.com,https://app.example.com";
  cors_methods = "GET,POST,OPTIONS";
  cors_headers = "Content-Type,Authorization";
}
```

### CORS Parameters
- `cors_enabled` - yes/no
- `cors_origins` - comma-separated allowed origins
- `cors_methods` - allowed HTTP methods
- `cors_headers` - allowed headers
- `cors_max_age` - preflight cache seconds

## Session Management

### Advertising Config
For inserting ads in streams:
```
stream with_ads {
  input rtsp://camera:554/stream;
  advertising {
    ad_duration = 30;
    ad_interval = 300;
    ad_server = "http://ad-server/request";
  };
}
```

### Geolocation Blocking
```
stream geo_restricted {
  input rtsp://camera:554/stream;
  auth token;
  geoip {
    blacklist = "CN,RU,KP";      # Block these countries
  };
}
```

### Auth Backend (on_play / on_publish)
Flussonic calls an external auth server when viewers connect:
```
stream custom_auth {
  input rtsp://camera:554/stream;
  on_play http://auth-server/verify;
  on_publish http://auth-server/verify;
}
```

**Auth Server receives the request with stream/token/IP info and responds:**
```
200 OK          (allow access)
403 Forbidden   (deny access)
```

## Troubleshooting

### Token Not Working
```bash
# Check auth backend configuration
curl -u user:pass http://server/streamer/api/v3/auth_backends

# Check active sessions
curl -u user:pass http://server/streamer/api/v3/sessions

# Test stream access with token
curl "http://server/stream/index.m3u8?token=YOUR_TOKEN"
```

### IP Blocked
```bash
# Check listener config
curl -u user:pass http://server/streamer/api/v3/config | jq '.listeners'

# Verify firewall
iptables -L | grep REJECT

# Check IP bans
grep -i "ban\|deny" /var/log/flussonic/flussonic.log
```

### DRM License Errors
- Verify license URL is accessible
- Check customer/user IDs
- Review DRM provider logs
- Test license URL: `curl -v https://license.server/`

### CORS Issues
```bash
# Test CORS headers
curl -i -H "Origin: https://player.example.com" \
  http://server/stream/index.m3u8

# Should include:
# Access-Control-Allow-Origin: https://player.example.com
```

### Commands
```bash
# Check auth config
curl -u user:pass http://server/streamer/api/v3/config | jq '.auth'

# List active sessions
curl -u user:pass http://server/streamer/api/v3/sessions

# Check CORS config
curl -u user:pass http://server/streamer/api/v3/config | jq '.cors'

# Test auth endpoint
curl -u user:pass http://server/streamer/api/v3/streams/stream1 | \
  jq '.auth'
```
