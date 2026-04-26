---
name: solace-performance-review
description: 'Solace performance audit and optimization guidance. Use for reviewing throughput, latency, client tuning, broker settings, message batching, flow control, and identifying performance bottlenecks.'
argument-hint: 'throughput, latency, tuning, or bottleneck'
---

# Solace Performance Review

Expert guidance for auditing and optimizing Solace PubSub+ performance.

## When to Use

- **Performance Audit**: Baseline and benchmark
- **Optimization**: Improve throughput/latency
- **Bottleneck Analysis**: Identify constraints
- **Tuning**: Client and broker optimization

## Performance Checklist

### Client-Side Performance

```markdown
## Client Performance Checklist

### Connection
- [ ] Connection keep-alive configured
- [ ] TCP_NODELAY enabled for low latency
- [ ] Appropriate send/receive buffer sizes
- [ ] Connection pooling (if applicable)

### Publishing
- [ ] Batching enabled for high throughput
- [ ] Async publishing where possible
- [ ] Appropriate QoS (direct vs guaranteed)
- [ ] Message size optimized

### Consuming
- [ ] Async message handling
- [ ] No blocking in callbacks
- [ ] Appropriate acknowledgment mode
- [ ] Flow control handled
```

### Broker-Side Performance

```markdown
## Broker Performance Checklist

### Resources
- [ ] Adequate CPU allocated
- [ ] Sufficient memory for spool
- [ ] Fast storage for guaranteed messaging
- [ ] Network bandwidth verified

### Configuration
- [ ] Max connections appropriately sized
- [ ] Spool limits configured
- [ ] Client profile limits set
- [ ] Event thresholds configured
```

## Performance Metrics Collection

### Key Metrics

```bash
#!/bin/bash
# Collect performance metrics

echo "=== Performance Metrics ==="

# Message rates
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default" | \
jq '{
  rxMsgRate: .data.rxMsgRate,
  txMsgRate: .data.txMsgRate,
  rxByteRate: .data.rxByteRate,
  txByteRate: .data.txByteRate,
  avgRxMsgSize: (if .data.rxMsgRate > 0 then .data.rxByteRate / .data.rxMsgRate else 0 end),
  avgTxMsgSize: (if .data.txMsgRate > 0 then .data.txByteRate / .data.txMsgRate else 0 end)
}'

# Connection stats
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default" | \
jq '{
  connections: .data.connectionCount,
  maxConnections: .data.maxConnectionCount,
  subscriptions: .data.subscriptionCount,
  maxSubscriptions: .data.maxSubscriptionCount
}'

# Spool stats
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default" | \
jq '{
  spoolUsageMB: (.data.msgSpoolUsage / 1048576 | floor),
  maxSpoolMB: .data.maxMsgSpoolUsage,
  spoolPercent: ((.data.msgSpoolUsage / 1048576) / .data.maxMsgSpoolUsage * 100 | floor),
  msgSpoolIngressMsgRate: .data.rxMsgRate,
  msgSpoolEgressMsgRate: .data.txMsgRate
}'
```

### Client Performance Stats

```bash
# Per-client statistics
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/clients" | \
jq '.data[] | {
  client: .clientName,
  rxMsgRate: .averageRxMsgRate,
  txMsgRate: .averageTxMsgRate,
  rxByteRate: .averageRxByteRate,
  txByteRate: .averageTxByteRate,
  slowSubscriber: .slowSubscriber
}'
```

### Queue Performance

```bash
# Queue ingress/egress rates
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | {
  queue: .queueName,
  bindCount: .bindCount,
  msgCount: .msgCount,
  ingressRate: .averageRxMsgRate,
  egressRate: .averageTxMsgRate,
  spoolUsage: .msgSpoolUsage,
  highWaterMark: .highMsgSpoolUsage
}'
```

## Common Performance Issues

### 1. Slow Subscribers

```bash
# Detect slow subscribers
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/clients" | \
jq '.data[] | select(.slowSubscriber == true) | {
  client: .clientName,
  user: .clientUsername,
  issue: "SLOW_SUBSCRIBER",
  eliding: .elidingEnabled
}'
```

**Solutions**:
- Enable message eliding
- Increase client buffer sizes
- Move processing off callback thread
- Scale consumers horizontally

### 2. Queue Depth Growing

```bash
# Find queues with growing backlog
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | select(.averageRxMsgRate > .averageTxMsgRate * 1.1) | {
  queue: .queueName,
  ingressRate: .averageRxMsgRate,
  egressRate: .averageTxMsgRate,
  backlog: .msgCount,
  issue: "INGRESS_EXCEEDS_EGRESS"
}'
```

**Solutions**:
- Add more consumers
- Optimize consumer processing
- Consider message batching
- Review message filtering

### 3. High Latency

**Client-side Tuning**:
```java
// Low-latency configuration
SessionProperties props = new SessionProperties();
props.setProperty(SessionProperties.TCP_NODELAY, "true");
props.setProperty(SessionProperties.SOCKET_SEND_BUF_SIZE, "65536");
props.setProperty(SessionProperties.SOCKET_RCV_BUF_SIZE, "65536");
props.setProperty(SessionProperties.KEEP_ALIVE_INTERVAL_MS, "3000");
```

### 4. Throughput Bottleneck

**Batching Configuration**:
```java
// High-throughput configuration
props.setProperty(SessionProperties.PUB_WINDOW_SIZE, "255");
props.setProperty(SessionProperties.SEND_BUFFER_SIZE, "1048576");

// Use async publishing
producer.setMessagePublishReceipt(receipt -> {
    // Handle asynchronously
});
```

## Performance Tuning Guide

### Client-Side Optimizations

| Setting | Low Latency | High Throughput |
|---------|-------------|-----------------|
| TCP_NODELAY | true | false |
| PUB_WINDOW_SIZE | 50 | 255 |
| SEND_BUFFER_SIZE | 65536 | 1048576 |
| Direct publish | ✅ | ✅ |
| Batching | ❌ | ✅ |

### Message Size Optimization

```python
# Compare serialization sizes
import json
import msgpack

data = {"orderId": "12345", "items": [...], "timestamp": "..."}

json_size = len(json.dumps(data))        # ~500 bytes
msgpack_size = len(msgpack.packb(data))  # ~300 bytes (40% smaller)
```

### Flow Control Handling

```java
// Handle flow control gracefully
producer.setFlowControlDisabled(false);
producer.setBackPressureHandler((message, unacked) -> {
    if (unacked > threshold) {
        // Slow down publishing
        Thread.sleep(10);
    }
    return true; // Continue
});
```

## Performance Benchmarking

### Baseline Test Script

```bash
#!/bin/bash
# Simple throughput test

# Start subscribers
for i in {1..3}; do
  sdkperf -cip=tcp://broker:55555 -cu=user@vpn -cp=pass \
    -sql=test-queue -cc=1 -mn=0 &
done

# Start publisher
sdkperf -cip=tcp://broker:55555 -cu=user@vpn -cp=pass \
  -pql=test-queue -mn=100000 -mr=10000

# Metrics to record:
# - Message rate (msgs/sec)
# - Latency (avg, p99, max)
# - CPU utilization
# - Spool usage
```

### Performance Report Template

```markdown
# Performance Audit Report

**System**: [name]
**Date**: [date]
**Test Duration**: [X hours]

## Summary
- Peak Throughput: [X] msgs/sec
- Average Latency: [Y] ms
- P99 Latency: [Z] ms
- Bottleneck: [identified issue]

## Metrics

### Message Rates
| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| Ingress | [X]/s | [Y]/s | ✅/❌ |
| Egress | [X]/s | [Y]/s | ✅/❌ |

### Latency
| Percentile | Value | Target |
|------------|-------|--------|
| p50 | [X] ms | [Y] ms |
| p95 | [X] ms | [Y] ms |
| p99 | [X] ms | [Y] ms |

### Resource Usage
| Resource | Peak | Limit | Headroom |
|----------|------|-------|----------|
| CPU | [X]% | 80% | [Y]% |
| Memory | [X] GB | [Y] GB | [Z] GB |
| Spool | [X]% | 80% | [Y]% |
| Connections | [X] | [Y] | [Z] |

## Bottleneck Analysis
[Description of identified bottlenecks]

## Optimization Recommendations
1. [Recommendation with expected impact]
2. [Recommendation with expected impact]

## Test Configuration
- Message size: [X] bytes
- Message type: [direct/guaranteed]
- Consumers: [X]
- Publishers: [X]
```

## Monitoring Dashboard Queries

### Prometheus Metrics

```yaml
# Key performance indicators
- name: solace_throughput_ingress
  query: rate(solace_vpn_rx_msg_count[5m])
  
- name: solace_throughput_egress
  query: rate(solace_vpn_tx_msg_count[5m])
  
- name: solace_queue_depth
  query: solace_queue_messages_spooled

- name: solace_slow_subscribers
  query: solace_client_slow_subscriber == 1
```

## Official Performance Best Practices (from Solace Documentation)

### G-1 Queue Tuning (Per-Client Egress Priority Queue)

```markdown
## Understanding Work Units

Each client has a G-1 queue (per-client egress priority queue) that buffers 
Guaranteed messages waiting for delivery or acknowledgment.

**Work Unit Definition:**
- 1 work unit = 2,048 bytes of message data
- Default G-1 queue max depth: 20,000 work units (~40 MB)

## Buffer Usage Formula

Total buffer usage (work units) = 
  (Number of Flows) × (Window Size per Flow) × (Avg Message Size in bytes) ÷ 2048
```

### Flow Configuration Impact

```bash
# PROBLEM: Client binds to 10 queues with window size 255 and avg 20KB messages
# Buffer usage = 10 × 255 × 20480 / 2048 = 25,500 work units
# EXCEEDS default 20,000 work units!

# SOLUTION 1: Reduce window size
# 10 × 25 × 20480 / 2048 = 2,500 work units (OK)

# SOLUTION 2: Increase min-msg-burst in client profile
# Set min-msg-burst ≥ sum of all flow window sizes
```

### Audit G-1 Queue Sizing

```bash
#!/bin/bash
# Calculate per-client buffer usage

echo "=== G-1 Queue Buffer Analysis ==="

# Get client flow information
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/clients" | \
jq '.data[] | {
  client: .clientName,
  flowCount: .flowCount,
  flowWindowSize: .flowWindowSize  
}' | while read -r client_info; do
  flow_count=$(echo "$client_info" | jq -r '.flowCount // 0')
  window_size=$(echo "$client_info" | jq -r '.flowWindowSize // 255')
  client=$(echo "$client_info" | jq -r '.client')
  
  # Estimate work units (assuming 10KB average message)
  avg_msg_bytes=10240
  work_units=$((flow_count * window_size * avg_msg_bytes / 2048))
  
  if [ "$work_units" -gt 20000 ]; then
    echo "WARNING: $client may exceed G-1 queue limit"
    echo "  Flows: $flow_count, Window: $window_size"
    echo "  Estimated work units: $work_units (limit: 20,000)"
  fi
done
```

### min-msg-burst Configuration

```bash
# If flow count × window size > default min-msg-burst, increase it
# This ensures NAB holds enough messages when client comes online

curl -X PATCH "$SEMP_URL/config/msgVpns/default/clientProfiles/default" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "allowGuaranteedMsgReceiveEnabled": true,
    "maxEndpointCountPerClientUsername": 100,
    "queueGuaranteedMinMsgBurst": 50000
  }'

# Formula: min-msg-burst >= sum of all flow window sizes for the client
# Example: 1000 endpoints × 50 window size = set min-msg-burst to 50000+
```

### TCP Buffer Sizing for WAN

```markdown
## Bandwidth-Delay Product

For WAN optimization, TCP buffers must be sized to fill the "pipe":

Buffer Size = Bandwidth × Round-Trip Time

Example: 1 Gbps link with 100ms RTT
Buffer = 1,000,000,000 bits/sec × 0.1 sec = 100,000,000 bits = 12.5 MB

## Platform-Specific Guidance

**Windows:** Receive buffer should be larger than send buffer (ratio 5:3)
- Send buffer: 90,000 bytes (default)
- Receive buffer: 150,000 bytes (default)

**Linux:** Configure system-wide TCP buffer limits:
/etc/sysctl.conf:
  net.core.rmem_max = 16777216
  net.core.wmem_max = 16777216
  net.ipv4.tcp_rmem = 4096 87380 16777216
  net.ipv4.tcp_wmem = 4096 65536 16777216
```

### Client Buffer Configuration

```java
// Java JCSMP
JCSMPChannelProperties channelProps = new JCSMPChannelProperties();
channelProps.setSendBuffer(262144);    // 256KB
channelProps.setReceiveBuffer(393216); // 384KB (larger for Windows)

properties.setProperty(JCSMPProperties.CLIENT_CHANNEL_PROPERTIES, channelProps);
```

```c
// C API
const char *sessionProps[] = {
    SOLCLIENT_SESSION_PROP_SOCKET_RCV_BUF_SIZE, "262144",
    SOLCLIENT_SESSION_PROP_SOCKET_SEND_BUF_SIZE, "131072",
    NULL
};
```

### Ultra-Low Latency Configuration

```markdown
## For Latency-Sensitive Applications

1. **Dispatch directly from I/O thread (JCSMP)**
   ```java
   properties.setBooleanProperty(JCSMPProperties.MESSAGE_CALLBACK_ON_REACTOR, true);
   // WARNING: Do NOT block in callbacks when using this
   ```

2. **Disable Nagle's algorithm**
   ```java
   channelProps.setTcpNoDelay(true);
   ```

3. **Use Direct messaging** (lower latency than Guaranteed)

4. **Pre-allocate and reuse messages** (avoid allocation overhead)

5. **Size AD window appropriately** (smaller windows = lower latency but lower throughput)
```

### Performance Tuning Checklist (Official)

```markdown
## Client-Side Tuning Checklist

### Connection
- [ ] TCP_NODELAY enabled for low latency
- [ ] Send/receive buffer sizes appropriate for network
- [ ] Keep-alive aligned with broker TCP keep-alive

### Guaranteed Messaging
- [ ] AD window size ≤ queue max-delivered-unacked-msgs-per-flow
- [ ] Total flow work units < 20,000 per client
- [ ] min-msg-burst sized for flow count × window size
- [ ] Client acknowledgment mode matches use case

### Publishing
- [ ] Non-blocking send for high throughput
- [ ] Batch sending where applicable (up to 50 messages)
- [ ] Message pre-allocation and reuse
- [ ] TTL set on time-sensitive messages

### Consuming
- [ ] No blocking in message callbacks
- [ ] Acknowledge messages promptly
- [ ] Handle redelivered messages idempotently

### Broker Side
- [ ] reject-msg-to-sender-on-discard enabled
- [ ] respect-ttl enabled on queues
- [ ] max-redelivery-count finite
- [ ] DLQ configured
```

## Reference Links

- **Performance Guide**: https://docs.solace.com/Best-Practices/Performance-Recommendations.htm
- **SDKPerf Tool**: https://docs.solace.com/SDKPerf/SDKPerf.htm
- **Tuning Guide**: https://docs.solace.com/Performance/Performance-Tuning.htm

## Further Reading

- [Benchmark Methodology](./references/benchmarking.md)
- [Client Tuning](./references/client-tuning.md)
- [Broker Sizing](./references/broker-sizing.md)
