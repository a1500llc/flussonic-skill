# Flussonic Retroview Monitoring Reference Guide

## Table of Contents
1. [Overview](#overview)
2. [Architecture & Core Concepts](#architecture--core-concepts)
3. [Input Monitoring (Priority)](#input-monitoring-priority)
4. [Server Dashboard Monitoring](#server-dashboard-monitoring)
5. [Alert Configuration](#alert-configuration)
6. [Integration with Prometheus/Grafana](#integration-with-prometheusgrafana)
7. [Configuration Examples](#configuration-examples)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Practical Tips for CDN Operators](#practical-tips-for-cdn-operators)

---

## Overview

**Retroview** is Flussonic's integrated monitoring service designed specifically for live streaming operations. It eliminates the need for external monitoring agents by providing native visibility into:
- Input stream health and status
- Server resource utilization
- DVR recording integrity
- Network performance across the entire streaming pipeline

**Key Principle:** "Input monitoring determines everything that will happen to the signal later" - prioritize input health above all metrics.

---

## Architecture & Core Concepts

### Monitoring Stack
- **No External Agents Required**: Built-in monitoring eliminates third-party dependencies
- **Time Range Flexibility**: Analyze issues from real-time to 30-day historical windows
- **Pattern Recognition**: Identify recurring issues vs. one-time failures
- **Integrated Alerting**: Email, Telegram, Slack, and webhook support

### Signal Flow in Context
1. **Input → Monitoring** (Retroview captures real-time stream quality)
2. **Processing → Metrics** (Transcoding, restreaming operations)
3. **Output → CDN Distribution** (Quality metrics feed back to monitoring)

---

## Input Monitoring (Priority)

### Why Input Monitoring Matters
Input stream health is the foundation. Errors here cascade through transcoding and restreaming. Early detection prevents downstream failures.

### Stream Status Categories

#### Critical: Stream Errors (Must Remain Zero)
Stream errors indicate packet loss, frame corruption, or encoding issues:

```
Error Types and Causes:
├── lost_packets    → SRT, RTSP, WebRTC, ST2110 packet loss (network issue)
├── ts_cc           → MPEG-TS continuity counter errors (typically network jitter)
├── ts_scrambled    → Encrypted streams (require CAM module configuration)
├── broken_payload  → Protocol-specific frame damage (partial data loss)
└── src_404/403/500 → HTTP source unavailable (fix source endpoint)
```

**Impact on Streaming:** Even one error can cause visible picture breakup, audio dropout, or subtitle loss.

#### Important: Stream Warnings (Server-Corrected)
Server handles these transparently, but they indicate stress:
- Timestamp anomalies
- Clock deviation issues
- Marker mode problems

#### Monitoring: DVR Recording Status
Critical for DVR operations:
- Write discontinuities
- Storage failures
- Performance degradation warnings

### Problem Stream Indication Graph
Displays most problematic streams over time. Use to:
- Identify daily patterns (e.g., 7 PM-1 AM failures = ISP peak congestion)
- Spot episodic failures (suggest capacity issues if simultaneous)
- Detect provider failures (sudden complete breakdowns)

### Pattern Recognition Examples

**Pattern 1: Evening Internet Peak (Recurring 7 PM-1 AM)**
```
Diagnosis: ISP traffic contention during prime hours
Action: Request increased bandwidth or upgrade SLA
Monitor: Check if pattern repeats on weekends (traffic-based) or weekdays (scheduled congestion)
```

**Pattern 2: Episodic Network Failures (Multi-Channel)**
```
Diagnosis: Simultaneous errors across multiple streams
Action: Check ingestion network capacity; scale input collection infrastructure
Monitor: Watch for gradual increase in frequency
```

**Pattern 3: Provider Failures (Sudden Complete Loss)**
```
Diagnosis: Source provider goes offline
Action: Contact provider; activate failover stream if configured
Monitor: Prevent cascade to CDN subscribers by alerting immediately
```

---

## Server Dashboard Monitoring

### CPU and Scheduler Load

**CPU Load Metric:**
- Threshold: 80% is elevated but not critical
- **Limitation:** Cannot predict linear growth by adding streams (non-linear scaling)
- Action: Monitor trend, not absolute value

**Scheduler Load (Higher Priority):**
- More reliable indicator of actual server capacity
- Threshold: 85% is warning level
- Action: Triggers alerts to prevent overload

### Memory Utilization
- Monitor total RAM usage (should remain stable)
- **Note:** System disables swap (not beneficial for streaming)
- Action: Scale horizontally when approaching 85%

### Disk Performance (Critical for DVR)

```
Metric              | Threshold | Action
────────────────────┼───────────┼──────────────────────
I/O Percentage      | < 80%     | Monitor real-time writes
Collapsed Writes    | Zero      | Early warning of storage lag
Failed Writes       | Zero      | CRITICAL: video loss risk
Fill Level          | Stable    | Add storage before 90%
Write Speed         | Trending  | Alert on sharp drops
Read Speed          | Trending  | Track quality over months
```

### Streams and Clients
- Graph active streams and connected clients
- Primary use: Monitor during system upgrades and scaling events

### Network Traffic
- Monitor incoming/outgoing for media server AND OS
- Key for identifying bottlenecks in ingestion vs. distribution

### GPU Status (Transcoding Operations)
- Monitor utilization and temperature
- **Critical:** GPU cooling is mandatory; prevent reaching 90°C
- Action: Throttle transcoding if temp approaches limit

---

## Alert Configuration

### Alert Types Reference

#### Server-Level Alerts

```
Alert Type         | Metric           | Threshold | Interval
─────────────────────┼──────────────────┼───────────┼─────────
CPU Load           | Average over 1h  | > 85%     | 10s-1h
Scheduler Load     | Average over 1h  | > 85%     | 10s-1h
Memory Usage       | Average over 1h  | > 85%     | 10s-1h
Disk I/O           | Average over 1h  | > 85%     | 10s-1h
```

#### Input Stream Alerts

```
Alert Type              | Trigger Condition        | Use Case
────────────────────────┼──────────────────────────┼────────────────────
streams_drop (Mass)     | > 10% streams offline    | Detect regional ISP issues
stream_dead             | Previously-working dead  | Catch provider failures
selected_stream_dead    | Monitor priority channel | High-value stream protection
flapping_streams        | > 3 offline/recovery in 3h | Unstable ingestion network
Offline % Change        | > 20-30% increase        | Threshold configurable
Bad Stream % Change     | > 15-25% increase        | Threshold configurable
```

### Creating an Alert (Step-by-Step)

1. **Select Target:**
   - Specific server or "All" servers (if supported)

2. **Select Scope:**
   - All streams OR specific stream(s)

3. **Choose Alert Type:**
   - Pick from server or input stream categories

4. **Configure Destination:**
   - Email, Telegram, Slack, or custom webhook

5. **Name the Alert:**
   - Example: `CDN-Input-04 Drop Detection`

6. **Set Evaluation Interval:**
   - Options: 10s, 30s, 1m, 5m, 1h
   - Faster = higher CPU cost; slower = delayed detection

7. **Confirm & Deploy**

### Contact Point Configuration

**Email Setup (Simple):**
```
Contact Name: CDN-Operations-Team
Email(s): ops@company.com,backup@company.com
```

**Advanced (Grafana Interface):**
- Telegram: Requires bot token + chat ID
- Slack: Requires webhook URL
- Webhooks: Custom HTTP endpoints for integration
- Requires Grafana's dedicated alerting interface

---

## Integration with Prometheus/Grafana

### Retroview → Grafana Workflow
1. Retroview exposes metrics natively (no exporter needed)
2. Prometheus scrapes Retroview endpoint
3. Grafana dashboards visualize metrics
4. Alerting rules evaluate conditions
5. Contact points route alerts

### Critical Metrics to Export
- Input stream error counters (lost_packets, ts_cc, broken_payload)
- CPU/scheduler/memory utilization
- Disk I/O and failed writes
- GPU temperature/utilization
- Active stream count
- Network throughput

---

## Configuration Examples

### Example 1: Protect High-Priority Stream from Drop

```yaml
Alert: Premium-Feed-Protection
Type: selected_stream_dead
Target Stream: "cdn-input-premium-1080p"
Evaluation Interval: 10s
Contact: ops@company.com
Threshold: Immediate (stream goes offline)
```

### Example 2: Detect Mass Input Failure

```yaml
Alert: Bulk-Input-Failure
Type: streams_drop (Mass)
Threshold: > 10% of streams offline
Evaluation Interval: 1m
Server: All
Contact: ops@company.com, slack-webhook
Action Trigger: If fires, page on-call engineer
```

### Example 3: Prevent Server Overload During Transcoding

```yaml
Alert: Transcoding-Overload-CPU
Type: scheduler_load
Threshold: > 85% for 1 hour
Evaluation Interval: 30s
Server: transcoding-01.production
Contact: ops@company.com
Escalation: Trigger load balancer to drain new streams
```

### Example 4: DVR Storage Critical

```yaml
Alert: DVR-Write-Failure
Type: disk_io
Threshold: > 85% I/O + failed writes > 0
Evaluation Interval: 10s
Server: dvr-storage-01
Contact: ops@company.com, slack-critical
Action: Trigger immediate storage expansion request
```

---

## Troubleshooting Guide

### Scenario 1: Recurring Evening Stream Errors (7 PM-1 AM)

**Diagnosis Steps:**
1. Open "Problem Stream Indication Graph" for 7-day view
2. Check if errors are simultaneous across multiple inputs
3. Correlate with ISP SLA; check if pattern includes weekends

**Solutions:**
- **If weekday-only:** ISP traffic contention → upgrade SLA or load balance across multiple ISPs
- **If every night:** Change ingestion time window or request ISP prioritization
- **If weekend too:** Structural ISP issue → switch providers or add redundant feed

### Scenario 2: Sudden "stream_dead" Alert

**Diagnosis Steps:**
1. Check Input Monitoring dashboard for affected stream
2. Verify network connectivity to source
3. Check source endpoint health (HTTP, RTSP, SRT logs)

**Quick Fixes:**
- Restart source encoder
- Verify authentication credentials
- Check firewall rules for source IP
- Validate codec compatibility

### Scenario 3: Flapping Streams (Repeated Offline/Recovery)

**Diagnosis Steps:**
1. Identify which streams are flapping (look for repeated dips in graph)
2. Check if timing correlates with server resource spikes
3. Verify network path stability (MTR, packet loss)

**Likely Causes:**
- Ingestion network instability (upgrade to dedicated link)
- Source encoder buffer issues (increase buffer, reduce bitrate)
- Server overload during encoding (scale transcoding)

### Scenario 4: Disk I/O Approaching 85%

**Diagnosis Steps:**
1. Check current disk fill level
2. Verify write speed trends (should be stable)
3. Confirm "collapsed writes" count (0 = good, >0 = storage lag)

**Actions:**
- Add storage immediately (don't wait for 90% full)
- Review DVR retention policies (reduce if possible)
- Implement tiered storage (SSD cache → HDD archive)
- Check for stuck transcode jobs consuming disk

### Scenario 5: GPU Temperature Nearing 90°C

**Diagnosis Steps:**
1. Monitor active transcode jobs
2. Check GPU utilization % vs. available vRAM
3. Verify cooling system operation

**Actions (Priority Order):**
1. **Immediate:** Reduce active transcode streams
2. **Short-term:** Add GPU cooling (fans, liquid cooling)
3. **Long-term:** Distribute transcoding across multiple GPUs

---

## Practical Tips for CDN Operators

### Best Practices for Input Monitoring

1. **Zero-Error Target:** Stream errors should trend to zero over time
   - Baseline acceptable errors → 0 errors (continuous improvement)
   - Keep error trend charts visible in NOC

2. **Priority Hierarchy:**
   - **Critical:** Stream errors, failed writes, GPU temperature
   - **High:** Scheduler load, memory usage, streams_drop alerts
   - **Medium:** Warnings, read speeds, flapping streams

3. **Alert Fatigue Prevention:**
   - Set evaluation intervals to 1m for non-critical alerts (reduce noise)
   - Use selected_stream_dead only for tier-1 channels
   - Group related alerts (e.g., "all-errors" vs. individual alert)

### CDN-Specific Configuration

For operators ingesting live signals → transcoding → CDN:

**Input Stage:**
- Monitor packet loss and encoder stability
- Alert on source provider changes or SLA violations
- Track input bitrate stability (variance = quality risk)

**Processing Stage:**
- Warn when scheduler load >75% (leave headroom)
- Monitor GPU temperature continuously
- Track transcode output bitrate consistency

**Output Stage:**
- Monitor CDN edge connection quality
- Ensure network link to CDN hub doesn't exceed 80% utilization
- Track delivery failures to subscribers (proxy metric)

### Automation Suggestions

1. **Auto-Scale Transcoding:** Trigger additional transcode servers when scheduler >75%
2. **Auto-Failover:** Switch to backup input when stream_dead alert fires
3. **Auto-Report:** Daily digest of error trends emailed to management
4. **Auto-Drain:** Gradually reduce new streams when server approaches threshold

---

## Quick Reference Checklist

- [ ] Input Monitoring dashboard visible in NOC at all times
- [ ] Stream errors alert configured for all priority channels
- [ ] CPU/Scheduler/Memory/Disk alerts set to 85% threshold
- [ ] DVD recording discontinuities monitored
- [ ] GPU temperature alert at 85°C
- [ ] Email contact point configured for operations team
- [ ] Flapping stream alert configured (3+ events in 3h)
- [ ] Mass failure alert set (>10% streams offline)
- [ ] Documentation of local ISP behavior patterns created
- [ ] Failover procedures tested for critical feeds

---

**Last Updated:** 2026-02-24
**Source:** Flussonic Retroview Official Documentation
**Target Users:** Streaming operators, CDN administrators, NOC engineers
