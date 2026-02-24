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

### Generate Token
```bash
curl -u admin:pass -X POST \
  http://localhost/api/v3/tokens \
  -H "Content-Type: application/json" \
  -d '{
    "name": "viewer1",
    "streams": ["camera1", "camera2"],
    "expires": 3600
  }'
```

### Response
```json
{
  "token": "abc123def456",
  "expires": "2024-02-25T15:30:00Z",
  "streams": ["camera1", "camera2"]
}
```

### Playback with Token
```
http://server/camera1/index.m3u8?token=abc123def456
http://server/camera1/index.ts?token=abc123def456
```

### Token Parameters
- `name` - token identifier
- `streams` - allowed streams (array)
- `expires` - expiration time (seconds or ISO timestamp)
- `description` - optional notes

### Token Management
```bash
# List tokens
curl -u admin:pass http://localhost/api/v3/tokens

# Revoke token
curl -u admin:pass -X DELETE \
  http://localhost/api/v3/tokens/abc123def456

# Refresh token
curl -u admin:pass -X PUT \
  http://localhost/api/v3/tokens/abc123def456 \
  -d '{"expires": 7200}'
```

## Secure Link

### Time-Limited URL
Generate URLs with expiration:

```bash
# Generate secure link
curl -u admin:pass -X POST \
  http://localhost/api/v3/secure-links \
  -d "stream=camera1&duration=3600&limit=1"
```

### Parameters
- `stream` - stream name
- `duration` - validity in seconds
- `limit` - max downloads (optional)
- `ip` - restrict to IP (optional)

### Playback URL
```
http://server/camera1/index.m3u8?secure_token=xyz&expires=1234567890
```

## DRM Systems

### Widevine Protection
```
stream protected_widevine {
  input rtsp://camera:554/stream;
  drm widevine {
    license_url = "https://license.server.com/widevine";
    customer_id = "YOUR_ID";
    user_id = "USER_123";
  };
}
```

### PlayReady Protection
```
stream protected_playready {
  input rtsp://camera:554/stream;
  drm playready {
    license_url = "https://license.server.com/playready";
    custom_data = "optional_metadata";
  };
}
```

### FairPlay Protection (Apple)
```
stream protected_fairplay {
  input rtsp://camera:554/stream;
  drm fairplay {
    license_url = "https://license.server.com/fairplay";
    certificate_url = "https://cert.server.com/cert";
  };
}
```

### Multi-DRM
```
stream multi_drm {
  input rtsp://camera:554/stream;
  drm widevine { license_url = "https://lic.server/widevine"; };
  drm playready { license_url = "https://lic.server/playready"; };
  drm fairplay { license_url = "https://lic.server/fairplay"; };
}
```

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

### Custom Auth Middleware
Flussonic can call external auth service:
```
stream custom_auth {
  input rtsp://camera:554/stream;
  auth middleware;
  middleware_url = "http://auth-server/verify";
}
```

**Auth Server Request:**
```
POST /verify HTTP/1.1
Host: auth-server
Content-Type: application/x-www-form-urlencoded

token=TOKEN&stream=STREAM&ip=IP&session_id=SID
```

**Response:**
```
200 OK          (allow access)
401 Unauthorized (deny access)
```

## Troubleshooting

### Token Not Working
```bash
# Check token exists
curl -u admin:pass http://localhost/api/v3/tokens | grep token_value

# Test with token
curl "http://localhost/api/v3/streams?token=YOUR_TOKEN"

# Check token expiration
curl -u admin:pass http://localhost/api/v3/tokens/TOKEN_ID
```

### IP Blocked
```bash
# Check listener config
curl http://localhost/api/v3/config | jq '.listeners'

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
curl http://localhost/api/v3/config | jq '.auth'

# List active tokens
curl -u admin:pass http://localhost/api/v3/tokens

# Check CORS config
curl http://localhost/api/v3/config | jq '.cors'

# Test auth endpoint
curl -u admin:pass http://localhost/api/v3/streams/stream1 | \
  jq '.auth'
```
