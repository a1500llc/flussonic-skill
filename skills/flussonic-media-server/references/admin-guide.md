# Flussonic Administration Guide

## Table of Contents
- [Installation](#installation)
- [System Requirements](#system-requirements)
- [Configuration](#configuration)
- [Security](#security)
- [Performance Tuning](#performance-tuning)
- [Licensing](#licensing)
- [Troubleshooting](#troubleshooting)

## Installation

### Ubuntu Installation
```bash
# Option 1: Download and review before executing
curl -sSf https://flussonic.com/public/install.sh -o install_flussonic.sh
less install_flussonic.sh   # Review the script contents
bash install_flussonic.sh
service flussonic start

# Option 2: Direct install (official method, less secure)
curl -sSf https://flussonic.com/public/install.sh | sh
service flussonic start
```

**Security note:** Piping remote scripts directly to `sh` bypasses review. Always prefer downloading and inspecting the script first (Option 1).

### RPM-based Systems (CentOS/RedHat)
```bash
cat > /etc/yum.repos.d/Flussonic.repo <<'YUMEOF'
[flussonic]
name=Flussonic
baseurl=https://apt.flussonic.com/rpm
enabled=1
gpgcheck=1
YUMEOF
yum -y install flussonic-erlang flussonic flussonic-transcoder
service flussonic start
```

**Security note:** Always use HTTPS for repository URLs and enable `gpgcheck=1` to verify package signatures.

### Activation
1. Start service: `service flussonic start`
2. Access UI: `http://FLUSSONIC-IP:80/`
3. Enter license key and set admin password

### Docker Deployment
```bash
# Sysadmin mode (for testing)
docker run -p 8081:80 -v /etc/flussonic:/etc/flussonic flussonic/flussonic

# With GPU support
docker run --rm -p 8081:80 -v /etc/flussonic:/etc/flussonic --gpus all \
  -e NVIDIA_DRIVER_CAPABILITIES='all' flussonic/flussonic
```

**DevOps mode:** Use environment variables
- STREAMER_HTTP - HTTP port
- STREAMER_EDIT_AUTH - credentials (user:password)
- STREAMER_CONFIG_EXTERNAL - external config backend
- LICENSE_KEY - license key

## System Requirements

Connections | CPU | RAM | Disk | Network
---|---|---|---|---
10 | Any | 128 MB | 40 MB | 100 Mbit/s
100 | Single core | 256 MB | 40 MB | 1 Gbit/s
1,000 | Quad core | 1 GB | 40 MB | 1 Gbit/s
5,000+ | Dual Xeon E5 | 16 GB | 40 MB | 10 Gbit/s

**OS:** Ubuntu LTS 24.04 or 22.04 (amd64, arm64)
**Browsers:** Firefox 70+, Chrome 79+
**Critical:** Disable swap completely

## Configuration

### Configuration File
Location: `/etc/flussonic/flussonic.conf`

### Essential Directives

**Admin Authentication:**
```
edit_auth admin password;
view_auth viewer viewpass;
```

**Listeners (HTTP/HTTPS ports):**
```
http 80 { }
https 443 {
  api false;
}
```

**TLS/SSL:**
- Certificate: `/etc/flussonic/streamer.crt`
- Private key: `/etc/flussonic/streamer.key`
- CA cert: `/etc/flussonic/streamer-ca.crt`

### Service Control
```bash
service flussonic start      # Start
service flussonic stop       # Stop
service flussonic restart    # Restart
service flussonic reload     # Reload (keeps connections)
service flussonic status     # Status check
```

## Security

### Access Control
- **edit_auth** - Full admin access
- **view_auth** - Read-only API access
- Password hashing supported (more secure)

### IP Whitelist
Configure in UI: Config > Listeners > Address field
Only specified IPs can access the UI

### API Access by Port
Disable API on specific ports:
```
https 443 { api false; }
```

### SSL Certificate Generation
```bash
cd /etc/flussonic
# Generate 4096-bit RSA key (minimum 2048-bit; never use 1024-bit)
openssl genrsa -out streamer.key 4096
chmod 600 streamer.key
openssl req -new -key streamer.key -out streamer.csr
openssl x509 -req -days 365 -in streamer.csr -signkey streamer.key -out streamer.crt
rm streamer.csr
```

**Security notes:**
- Use at least 2048-bit RSA keys (4096 recommended). 1024-bit is considered insecure.
- Set `chmod 600` on private keys to restrict access to root only.
- For production, use Let's Encrypt or a trusted CA instead of self-signed certificates.

### Let's Encrypt Certificates (Recommended for Production)
Flussonic has built-in Let's Encrypt support. Configure in UI under TLS settings, or via the private API:
```bash
curl -u user:pass -X POST http://server/streamer/api/v3/tls/letsencrypt
```

## Performance Tuning

### Critical Settings
1. Disable swap - Cannot extend RAM with swap for streaming
2. Set max open files: Check ulimit and increase if needed
3. NTP sync - Keep system time synchronized

### Fine-tuning Options
- Monitor CPU/memory usage during peak load
- Consider clustering for 5000+ concurrent connections
- Use SSD storage for VOD files
- Network should be 10 Gbit/s for 5000+ connections

### Logs Location
```
/var/log/flussonic/
```

## Licensing

### License Key
- Obtained from: my.flussonic.com
- Online activation requires internet connectivity
- Offline activation available via `GET /streamer/api/v3/license/request` (generates a request file for manual activation)
- License tied to hardware MAC address
- CPU and connection limits based on license tier

### Activation
- Enter license during first-run UI setup
- Or configure in `/etc/flussonic/flussonic.conf`
- Check license status in UI: Config > Settings > License
- For air-gapped environments, use the offline activation API endpoint

## Troubleshooting

### Common Issues

**Port 80 already in use**
```bash
netstat -tlnp | grep :80
```

**Installation fails**
- Verify 64-bit Ubuntu (LTS 24.04 or 22.04)
- Check /var/log/flussonic/ for errors
- Ensure HTTP port 80 is free

**Cannot access UI**
- Verify service is running: `service flussonic status`
- Check firewall rules
- Verify IP address in browser URL
- Check listener configuration for IP restrictions

**High memory usage**
- Check number of concurrent streams
- Review transcode settings
- Monitor: `ps aux | grep flussonic`

**Streams not playing**
- Verify source URL is accessible
- Check firewall on server
- Review stream logs in UI

### Debug Commands
```bash
service flussonic status
tail -f /var/log/flussonic/flussonic.log
service flussonic reload
```
