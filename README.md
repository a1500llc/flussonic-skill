# Flussonic Media Server Skill

A comprehensive Claude AI skill for Flussonic Media Server — covering configuration, API, live streaming, transcoding, DVR, restreaming, CDN, protocols, monitoring, and troubleshooting.

## Installation

```bash
npx skills add https://github.com/YOUR_USERNAME/YOUR_REPO
```

Or install to a specific agent:

```bash
npx skills add https://github.com/YOUR_USERNAME/YOUR_REPO --skill flussonic-media-server -a claude-code
```

## What's Included

This skill provides expert-level knowledge on:

- Live stream ingest (RTMP, SRT, WebRTC, multicast, RTSP, NDI)
- Transcoding with hardware acceleration (NVENC, QSV)
- DVR recording and playback with configurable retention
- Multi-protocol delivery (HLS, LL-HLS, DASH, WebRTC, MSE)
- Restreaming and CDN push (YouTube, Facebook, Twitch, custom)
- Clustering, load balancing, and CDN peering
- Authorization, DRM (Widevine, PlayReady, FairPlay), secure links
- Retroview input monitoring, alerting, Prometheus/Grafana integration
- Full API reference (74 control endpoints + 53 streaming endpoints)
- Server internals, Lua scripting, contrib diagnostic tools

## Skill Structure

```
skills/
  flussonic-media-server/
    SKILL.md                          # Main skill (architecture, workflows, routing)
    references/
      admin-guide.md                  # Installation, performance, firewall
      ingest-sources.md               # RTMP/SRT/WebRTC/multicast ingest
      transcoding.md                  # NVENC, QSV, multibitrate, overlays
      dvr-recording.md                # DVR config, retention, nPVR, RAID
      playback-delivery.md            # HLS/DASH/LL-HLS/WebRTC/MSE
      vod.md                          # VOD files, SMIL, cloud
      push-restream.md                # CDN push, social media, SRT push
      cluster-cdn.md                  # Clustering, load balancing, peering
      auth-drm.md                     # Tokens, DRM, GeoIP, session limits
      protocols.md                    # RTMP/RTSP/SRT/WebRTC/ONVIF details
      iptv-ott.md                     # IPTV, EPG, ad insertion
      api-reference-endpoints.md      # 74 control API endpoints
      api-streaming-endpoints.md      # 53 streaming endpoints
      general-concepts.md             # Data model, glossary, architecture
      retroview-monitoring.md         # Input monitoring, dashboards, alerts
      server-internals.md             # Directory structure, contrib tools, Lua, private API
```

## Sources

Built from official Flussonic documentation (148 pages), OpenAPI 3.1.0 specs (127 endpoints, 563 schemas), and live server exploration (v26.02).
