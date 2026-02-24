# Flussonic Clustering & CDN Guide

## Table of Contents
- [Cluster Overview](#cluster-overview)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [Cluster Ingest](#cluster-ingest)
- [Load Balancing](#load-balancing)
- [Transcoding Cluster](#transcoding-cluster)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)

## Cluster Overview

Clustering distributes streaming load across multiple servers:
- Horizontal scalability (5000+ connections)
- High availability and failover
- Load distribution
- Redundant transcoding
- Centralized management

## Architecture

### Cluster Topology
```
             Internet (Viewers)
                  |
    +-----------+-+-----------+
    |           |             |
  Server1     Server2       Server3
  (Pull)      (Pull)        (Pull)
    |           |             |
    +-----------+-+-----------+
                  |
         Source Stream (Grabber)
         (IP Camera, Satellite, etc.)
```

### Components
- **Grabber:** Captures/ingests source
- **Cluster Peers:** Pull from grabber, serve to viewers
- **Cluster Key:** Shared secret across peers

## Configuration

### Basic Cluster Setup
All peers (grab + stream servers):

```
cluster_key shared-secret-key;

peer server1.example.com;
peer server2.example.com;
peer server3.example.com;

stream camera1 {
  input rtsp://192.168.1.100:554/stream;
  cluster_ingest;
}
```

### Designated Grabber
Single server ingests, others pull:

**Grabber:**
```
cluster_key shared-key;
peer server1;
peer server2;

stream camera1 {
  input rtsp://camera:554/stream;
}
```

**Server 1 & 2:**
```
cluster_key shared-key;
peer grabber;
peer server1;
peer server2;

stream camera1 {
  input m4f://grabber/camera1;
}
```

**URL format:** `m4f://PEER_HOSTNAME/STREAM_NAME`

## Cluster Ingest

### Stream Capture Distribution
Distributes stream capture across peers:

```
stream dvb_stream {
  input mpts-dvb://a1?program=1001;
  cluster_ingest;
}
```

**Behavior:**
- Only one peer captures stream
- Others receive via cluster
- Automatic failover if peer dies
- Load rebalances on failure

### Multistream Setup
```
stream stream01 { input rtsp://cam1:554/s; cluster_ingest; }
stream stream02 { input rtsp://cam2:554/s; cluster_ingest; }
stream stream03 { input rtsp://cam3:554/s; cluster_ingest; }
stream stream04 { input rtsp://cam4:554/s; cluster_ingest; }
```

Each stream handled by one peer, distributed across cluster.

## Load Balancing

### Viewer Load Distribution
```
HTTP LB (DNS/IP-based)
         |
    +----+----+
    |    |    |
  S1   S2    S3
    |    |    |
  +----+----+
    Cluster
```

### DNS Round-Robin
```bash
# Create DNS records
stream.example.com IN A 192.168.1.10
stream.example.com IN A 192.168.1.11
stream.example.com IN A 192.168.1.12
```

### IP-Based Load Balancing
Use external LB (Nginx, HAProxy):

**Nginx:**
```nginx
upstream flussonic {
  server server1:80;
  server server2:80;
  server server3:80;
}

server {
  listen 80;
  location / {
    proxy_pass http://flussonic;
  }
}
```

### Geographic Distribution (CDN)
- Place servers in multiple regions
- Use DNS geolocation
- Route viewers to nearest server
- Reduces latency

## Transcoding Cluster

### Redundant Transcoding
Multiple transcoders handle streams:

**Config (4 transcoders):**
```
cluster_key abcd;
peer transcoder1;
peer transcoder2;
peer transcoder3;
peer transcoder4;

stream dvb01 {
  input m4f://grabber1/dvb01;
  transcoder vb=1000k deinterlace=true ab=128k;
  cluster_ingest;
}
```

**Scenario:** 200 channels from 2 grabbers, transcoded by 4 servers
- Each transcoder handles ~50 streams
- If one dies, others take over
- Load redistributes automatically

### Peer Failure Handling
When transcoder fails:
- Its streams reassigned to healthy peers
- Others temporarily overloaded (60-70% to 90%)
- Automatic recovery when peer comes back online

## Monitoring

### Cluster Status
```bash
# Check peer status
curl http://localhost/api/v3/peers

# Get cluster key info
curl http://localhost/api/v3/config | grep cluster
```

### Stream Distribution
```bash
# See which peer handles each stream
curl http://localhost/api/v3/streams | jq '.[] | {name, peer}'
```

### Health Check
```bash
# Monitor cluster connectivity
watch -n 5 'curl -s http://localhost/api/v3/system/info | jq .cluster'
```

### Logs
```bash
tail -f /var/log/flussonic/flussonic.log | grep -i cluster
```

## Troubleshooting

### Peers Not Connecting
```bash
# Verify DNS resolution
nslookup peer-hostname

# Test connectivity
nc -zv peer-hostname 1024

# Check firewall
iptables -L | grep 1024

# Verify cluster_key matches across all peers
grep cluster_key /etc/flussonic/flussonic.conf
```

### Streams Not Pulled from Grabber
- Verify peer name matches
- Check m4f:// URL format
- Confirm stream exists on grabber: `curl http://grabber/api/v3/streams`
- Review logs for connection errors

### Uneven Load Distribution
- Check stream distribution: `curl http://localhost/api/v3/streams | jq`
- May be intentional based on cluster_ingest assignment
- Monitor CPU usage per peer
- Consider rebalancing manually via config

### Failover Not Working
- Verify all peers in config: `grep "^peer" /etc/flussonic/flussonic.conf`
- Check network connectivity between peers
- Ensure cluster_key is identical
- Review cluster logs

### High CPU on Some Peers
- Unequal stream distribution
- Check: `curl http://localhost/api/v3/system/info | jq .cpu`
- Redistribute streams via config
- Monitor CPU per stream

### Commands
```bash
# Detailed stream info
curl http://localhost/api/v3/streams/stream1 | jq

# Cluster configuration
curl http://localhost/api/v3/config | jq '.cluster'

# List all peers
curl http://localhost/api/v3/peers | jq

# Reload cluster config
service flussonic reload

# Check cluster network
ping -c 3 peer-hostname
```

### Performance Tips
- Use dedicated cluster network (private network)
- Keep cluster_key secure
- Monitor inter-peer latency
- Balance grabbers and stream servers
- Use SSD for local caching
- Monitor disk for DVR/cache
