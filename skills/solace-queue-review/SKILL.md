---
name: solace-queue-review
description: 'Solace queue configuration audit. Use for reviewing queue settings, spool limits, access types, subscription bindings, dead letter queue setup, and identifying queue configuration anti-patterns.'
argument-hint: 'audit or configuration'
---

# Solace Queue Review

Expert guidance for auditing Solace queue configurations against best practices.

## When to Use

- **Queue Audit**: Review existing queue configurations
- **Capacity Planning**: Validate spool settings
- **HA Review**: Check queue resilience
- **Performance**: Identify bottlenecks

## Queue Audit Checklist

### Basic Configuration

```markdown
## Queue Configuration Checklist

### Naming
- [ ] Descriptive, meaningful names
- [ ] Consistent naming convention
- [ ] No special characters
- [ ] Reflects purpose/owner

### Access Type
- [ ] Appropriate access type selected
  - Exclusive: Single consumer
  - Non-exclusive: Competing consumers
- [ ] Access type matches use case

### Permissions
- [ ] Owner specified for queue
- [ ] Permission level appropriate
- [ ] No open "delete" permission
```

### Spool Configuration

```markdown
## Spool Management Checklist

### Sizing
- [ ] Max spool size appropriate for workload
- [ ] Not over-provisioned (wasting resources)
- [ ] Not under-provisioned (risk of message loss)
- [ ] VPN spool limits considered

### TTL and Expiry
- [ ] TTL enabled if messages are time-sensitive
- [ ] Reasonable TTL values (not too long)
- [ ] Respect TTL enabled on queue

### Quotas
- [ ] Quota events configured
- [ ] Alerts on high spool usage
- [ ] Monitoring in place
```

### Dead Letter Queue

```markdown
## DLQ Configuration Checklist

- [ ] DLQ configured for important queues
- [ ] DLQ has adequate spool
- [ ] DLQ monitoring enabled
- [ ] Process to handle DLQ messages
- [ ] Max redelivery count set appropriately
```

## Automated Queue Audit

### Gather Queue Metrics

```bash
#!/bin/bash
# Comprehensive queue audit

echo "=== Queue Configuration Audit ==="

# Get all queue configurations
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | {
  name: .queueName,
  accessType: .accessType,
  maxSpool: .maxMsgSpoolUsage,
  respectTtl: .respectTtlEnabled,
  dlq: .deadMsgQueue,
  maxRedelivery: .maxRedeliveryCount,
  owner: .owner,
  permission: .permission
}'
```

### Check Spool Usage

```bash
#!/bin/bash
# Find queues approaching spool limits

echo "=== Spool Usage Report ==="

curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | {
  name: .queueName,
  spoolUsageMB: (.msgSpoolUsage / 1048576 | floor),
  maxSpoolMB: .maxMsgSpoolUsage,
  usagePercent: ((.msgSpoolUsage / 1048576) / .maxMsgSpoolUsage * 100 | floor),
  msgCount: .msgCount
} | select(.usagePercent > 50)' | \
jq -s 'sort_by(-.usagePercent)'
```

### Identify Stale Queues

```bash
#!/bin/bash
# Find queues with no consumers and accumulated messages

curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | select(.bindCount == 0 and .msgCount > 0) | {
  name: .queueName,
  bindCount: .bindCount,
  msgCount: .msgCount,
  oldestMsgAge: .oldestMsgAgeInSeconds,
  status: "ORPHANED"
}'
```

### Missing DLQ Configuration

```bash
#!/bin/bash
# Find queues without DLQ

curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | select(.deadMsgQueue == null or .deadMsgQueue == "") | {
  name: .queueName,
  issue: "No DLQ configured"
}'
```

## Queue Anti-Patterns

### 1. Oversized Spool

```bash
# Anti-pattern: Queue with huge spool that's rarely used
# ISSUE: Wastes resources, hides problems
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | select(.maxMsgSpoolUsage > 10000 and .msgSpoolUsage < 100) | {
  name: .queueName,
  maxSpool: .maxMsgSpoolUsage,
  actualUsage: .msgSpoolUsage,
  issue: "Over-provisioned spool"
}'
```

### 2. Missing DLQ

```bash
# Anti-pattern: Production queue without dead letter handling
# ISSUE: Poison messages block processing
# RECOMMENDATION: Configure DLQ with max redelivery count
```

### 3. Wrong Access Type

```bash
# Anti-pattern: Non-exclusive queue for ordered processing
# ISSUE: Message ordering lost with competing consumers
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | select(
  .accessType == "non-exclusive" and 
  (.queueName | test("order|sequence|serial|fifo"))
) | {
  name: .queueName,
  issue: "Non-exclusive queue may not preserve order"
}'
```

### 4. No TTL

```bash
# Anti-pattern: Time-sensitive messages without TTL
# ISSUE: Stale messages processed, resources wasted
# CHECK for queues with old messages and no TTL:
curl -s -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/default/queues" | \
jq '.data[] | select(.oldestMsgAgeInSeconds > 86400) | {
  name: .queueName,
  oldestMsgHours: (.oldestMsgAgeInSeconds / 3600 | floor),
  issue: "Messages older than 24 hours"
}'
```

### 5. Excessive Redelivery

```bash
# Anti-pattern: High redelivery count or unlimited
# ISSUE: Poison messages loop forever
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | select(.maxRedeliveryCount > 10 or .redeliveryEnabled == false) | {
  name: .queueName,
  maxRedelivery: .maxRedeliveryCount,
  issue: "Excessive redelivery limit"
}'
```

## Queue Scoring

### Scoring Criteria

| Category | Weight | Good | Warning | Bad |
|----------|--------|------|---------|-----|
| Spool sizing | 20% | ≤50% used | 50-80% | >80% |
| DLQ config | 20% | Configured | Partial | None |
| TTL | 15% | Enabled | - | Disabled |
| Redelivery | 15% | 1-5 | 6-10 | >10 |
| Access type | 15% | Matches use | - | Mismatch |
| Consumer binding | 15% | Active | 0 w/msgs | - |

### Calculate Queue Score

```python
def score_queue(queue_config, queue_monitor):
    score = 100
    
    # Spool usage
    usage_pct = queue_monitor['msgSpoolUsage'] / (queue_config['maxMsgSpoolUsage'] * 1048576) * 100
    if usage_pct > 80:
        score -= 20
    elif usage_pct > 50:
        score -= 10
    
    # DLQ configuration
    if not queue_config.get('deadMsgQueue'):
        score -= 20
    
    # TTL
    if not queue_config.get('respectTtlEnabled'):
        score -= 15
    
    # Redelivery
    if queue_config.get('maxRedeliveryCount', 0) > 10:
        score -= 15
    
    # Consumer binding
    if queue_monitor.get('bindCount', 0) == 0 and queue_monitor.get('msgCount', 0) > 0:
        score -= 15
    
    return max(0, score)
```

## Queue Review Report Template

```markdown
# Queue Configuration Audit Report

**VPN**: [name]
**Date**: [date]
**Auditor**: [name]

## Summary
- Queues Reviewed: [count]
- Healthy: [count] ([%])
- Warning: [count] ([%])
- Critical: [count] ([%])

## Queue Inventory
| Queue | Access | Spool (MB) | Usage (%) | DLQ | Score |
|-------|--------|------------|-----------|-----|-------|
| [name] | Exclusive | 1000 | 45% | ✅ | 85 |

## Critical Issues
| Queue | Issue | Impact | Recommendation |
|-------|-------|--------|----------------|
| [name] | [issue] | [impact] | [fix] |

## Spool Analysis
- Total Spool Allocated: [X] GB
- Total Spool Used: [Y] GB
- Over-provisioned: [count] queues
- Near capacity: [count] queues

## DLQ Status
- Queues with DLQ: [count]/[total]
- DLQ with messages: [count]
- Messages in DLQs: [total]

## Recommendations
1. Configure DLQ for critical queues
2. Reduce spool for over-provisioned queues
3. Enable TTL for time-sensitive queues
4. Add consumers to orphaned queues
```

## Best Practice Configuration

```bash
# Recommended queue configuration
curl -X POST "$SEMP_URL/config/msgVpns/default/queues" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "order-processing",
    "accessType": "exclusive",
    "maxMsgSpoolUsage": 1000,
    "respectTtlEnabled": true,
    "maxTtl": 86400,
    "redeliveryEnabled": true,
    "maxRedeliveryCount": 3,
    "deadMsgQueue": "#dead-letter-queue",
    "permission": "consume",
    "owner": "order-service",
    "egressEnabled": true,
    "ingressEnabled": true
  }'
```

## Official Best Practices (from Solace Documentation)

### AD Window Size vs max-delivered-unacked-msgs-per-flow

```markdown
## Critical Configuration Coordination

The Assured Delivery (AD) window size configured on the client API should NOT be
greater than the `max-delivered-unacked-msgs-per-flow` value set on the queue.

| Setting | Where | Purpose |
|---------|-------|---------|
| AD Window Size | Client API (flow property) | Messages in transit between broker and client |
| max-delivered-unacked-msgs-per-flow | Queue (broker config) | Messages broker can deliver without acknowledgment |

**Problem:** If AD window > max-unacked, the API cannot acknowledge messages in a
timely manner because the broker limits delivery anyway.

**Recommendation:** Set AD window size ≤ max-delivered-unacked-msgs-per-flow
```

### Single-Message Processing (Strict Ordering)

```yaml
# For strict one-at-a-time message processing:
queue_config:
  max-delivered-unacked-msgs-per-flow: 1

client_flow:
  window_size: 1

# This ensures consistent delivery timing
# Without this, ACK threshold timing can cause reception delay variation
```

### Verify AD Window Coordination

```bash
#!/bin/bash
# Check for AD window size mismatches

echo "=== AD Window Size Coordination Check ==="

# Get queue max-delivered-unacked-msgs-per-flow settings
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | {
  queue: .queueName,
  maxDeliveredUnacked: .maxDeliveredUnackedMsgsPerFlow
}'

# Compare against typical client AD window sizes (default: 255)
# Flag queues where max-unacked < typical AD window
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/queues" | \
jq '.data[] | select(.maxDeliveredUnackedMsgsPerFlow < 255) | {
  queue: .queueName,
  maxDeliveredUnacked: .maxDeliveredUnackedMsgsPerFlow,
  warning: "Client AD window (default 255) may exceed this limit"
}'
```

### Temporary Endpoint Spool Size

```markdown
## Caution with Temporary Endpoints

By default, temporary endpoints have large spool quotas:
- Appliance: 4000 MB
- Software broker: 1500 MB

If applications create many temporary endpoints, spool can be exhausted quickly.

**Recommendation:** Override default spool size when creating temporary endpoints:
```

```java
// Java - reduce temporary queue spool
EndpointProperties endpointProps = new EndpointProperties();
endpointProps.setMaxMsgSize(10000000);  // 10MB
endpointProps.setQuota(100);  // 100MB instead of 1500MB

session.provision(tempQueue, endpointProps, JCSMPSession.FLAG_IGNORE_ALREADY_EXISTS);
```

### Max Redelivery Configuration

```bash
# RECOMMENDATION: Set finite max redelivery to prevent poison message loops
# Default is infinite redelivery

curl -X PATCH "$SEMP_URL/config/msgVpns/default/queues/order-processing" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "maxRedeliveryCount": 5,
    "redeliveryEnabled": true,
    "deadMsgQueue": "#DEAD_MSG_QUEUE"
  }'

# Benefits:
# - Poison messages eventually move to DLQ instead of blocking
# - Resources freed for processing good messages
# - Failed message can be examined in DLQ
```

### reject-msg-to-sender-on-discard

```bash
# RECOMMENDATION: Enable unless you have specific reason not to

curl -X PATCH "$SEMP_URL/config/msgVpns/default/queues/order-processing" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "rejectMsgToSenderOnDiscardBehavior": "always"
  }'

# When enabled:
# - Client receives NACK when messages are discarded
# - Allows client to take corrective action (pause publishing, etc.)
#
# Consider DISABLING when:
# - Multiple queues subscribe to same topic
# - Other queues should continue receiving even if one queue is full
```

### Queue Sizing Audit Checklist

```markdown
## AD Window & Message Flow Checklist

- [ ] AD window size ≤ max-delivered-unacked-msgs-per-flow for all flows
- [ ] For ordered processing: both set to 1
- [ ] Temporary endpoint spool sizes overridden if many are created
- [ ] max-redelivery-count is finite (3-10 recommended)
- [ ] DLQ configured for max-redelivery target
- [ ] reject-msg-to-sender-on-discard enabled unless multi-queue fanout
- [ ] respect-ttl enabled if TTL is used
```

## Reference Links

- **Queue Configuration**: https://docs.solace.com/Messaging/Guaranteed-Messaging/Configuring-Queues.htm
- **Dead Letter Queues**: https://docs.solace.com/Messaging/Guaranteed-Messaging/Message-Redelivery.htm
- **Best Practices**: https://docs.solace.com/Best-Practices/Queue-Best-Practices.htm

## Further Reading

- [Capacity Planning](./references/capacity-planning.md)
- [DLQ Handling](./references/dlq-patterns.md)
- [Performance Tuning](./references/queue-performance.md)
