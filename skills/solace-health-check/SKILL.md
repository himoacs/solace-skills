---
name: solace-health-check
description: 'Comprehensive Solace health check and best practices audit. Use for holistic application reviews, production readiness assessments, security posture checks, performance audits, and generating prioritized remediation reports. Invokes specialized review skills as needed.'
argument-hint: 'full, security, performance, or production-readiness'
---

# Solace Health Check

Master audit skill for comprehensive Solace implementation reviews. Performs holistic assessment across all dimensions and generates prioritized findings.

## When to Use

- **Full Health Check**: Complete assessment of Solace implementation
- **Production Readiness**: Verify deployment is production-ready
- **Security Audit**: Focus on security posture
- **Performance Review**: Identify optimization opportunities
- **Pre-Go-Live**: Final validation before launch

## Assessment Framework

### Dimensions Evaluated

| Dimension | Weight | Aspects |
|-----------|--------|---------|
| **Security** | 25% | Authentication, ACLs, TLS, credentials |
| **Reliability** | 20% | HA config, error handling, reconnection |
| **Performance** | 20% | QoS selection, batching, connection patterns |
| **Observability** | 15% | Monitoring, logging, tracing |
| **Design** | 15% | Topic architecture, schema design, patterns |
| **Operations** | 5% | Deployment, backup, procedures |

### Severity Levels

| Level | Description | Action Required |
|-------|-------------|-----------------|
| **Critical** | Security vulnerability or data loss risk | Immediate fix before production |
| **High** | Significant issue affecting reliability/performance | Fix within current sprint |
| **Medium** | Best practice violation with moderate impact | Plan for near-term fix |
| **Low** | Minor improvement opportunity | Consider for future |

## Health Check Procedure

### Step 1: Gather Context

Collect information about the implementation:

```markdown
## Implementation Context

**Application Type**: [ ] Microservice [ ] Integration [ ] Real-time analytics [ ] IoT
**Environment**: [ ] Development [ ] Staging [ ] Production
**Broker Type**: [ ] Solace Cloud [ ] Software Broker [ ] Appliance
**Languages/Frameworks**: _______________
**Expected Message Rate**: _____ msg/sec
**Message Size (avg)**: _____ KB
**SLA Requirements**: _______________
```

### Step 2: Code Review

Review application code for:

#### Connection Management
```
✓ Session lifecycle properly managed
✓ Reconnection strategy configured (official: 5+ min duration)
  - connect_retries: 1
  - reconnect_retries: 20 (or -1 for replication)
  - reconnect_retry_wait: 3000ms
  - connect_retries_per_host: 5
✓ Graceful shutdown implemented
✓ Connection pooling appropriate
✓ Keep-alive interval matches broker TCP keep-alive (~3s default)
```

#### Error Handling
```
✓ All error events have handlers (session + flow events)
✓ Retry logic for recoverable errors
✓ Dead-letter queue configured with finite max-redelivery (3-5)
✓ Errors logged with context
✓ Handles redelivered messages idempotently (flag persists during HA, not DR)
✓ Handles unexpected message formats (always ack to prevent infinite loops)
```

#### Message Processing
```
✓ Acknowledgments AFTER successful processing
✓ Idempotent processing (handles duplicate deliveries)
✓ Ordering requirements met (exclusive queue if needed)
✓ No blocking in callbacks (dispatch to worker threads)
✓ Pre-allocate and reuse messages for high throughput
```

### Step 3: Configuration Review

Review broker and application configuration:

#### Queue Configuration
```
✓ Appropriate access type (exclusive/non-exclusive)
✓ Max message size set correctly
✓ Spool quota appropriate
✓ Max redelivery count > 0 (recommend 3-5)
✓ DMQ configured for critical queues
✓ reject-msg-to-sender-on-discard enabled
✓ respect-ttl enabled if TTL used
✓ AD window size ≤ max-delivered-unacked-msgs-per-flow
```

#### Security Configuration
```
✓ No default credentials
✓ ACL profiles follow least privilege
✓ TLS enabled for production
✓ Client certificates for sensitive apps
✓ No secrets in code/logs
```

### Step 4: Architecture Review

Review design decisions:

#### Topic Architecture
```
✓ Hierarchical topic design (Domain/Noun/Verb/Version/Properties)
✓ Consistent naming conventions (camelCase preferred, kebab-case OK)
✓ No overly broad wildcards (never subscribe to `>` alone)
✓ Version segments where needed (v1, v2 format)
✓ Multi-tenant isolation
✓ No environment names in topics (dev/qa/prod)
✓ No tracing info in topics (use Distributed Tracing)
✓ No message selectors for filtering (use topic hierarchy instead)
```

#### Messaging Patterns
```
✓ Appropriate QoS selection
✓ Request-reply timeout handling
✓ Transaction scope appropriate
✓ Replay strategy defined
```

### Step 5: Operations Review

Review operational readiness:

#### Monitoring
```
✓ Health checks configured
✓ Key metrics exposed (Prometheus/etc)
✓ Alerts for critical thresholds
✓ Dashboard for operations
```

#### Documentation
```
✓ Topic taxonomy documented
✓ Queue ownership defined
✓ Runbooks for common issues
✓ On-call procedures
```

## Report Template

```markdown
# Solace Health Check Report

**Application**: [Application Name]
**Date**: [Assessment Date]
**Assessor**: [Name/System]
**Environment**: [Dev/Staging/Prod]

---

## Executive Summary

**Overall Score**: XX/100
**Production Ready**: [ ] Yes [ ] No - requires remediation

| Category | Score | Status |
|----------|-------|--------|
| Security | XX/25 | 🔴/🟡/🔵 |
| Reliability | XX/20 | 🔴/🟡/🔵 |
| Performance | XX/20 | 🔴/🟡/🔵 |
| Observability | XX/15 | 🔴/🟡/🔵 |
| Design | XX/15 | 🔴/🟡/🔵 |
| Operations | XX/5 | 🔴/🟡/🔵 |

---

## Critical Findings (Must Fix)

### [SECURITY-001] Default Credentials in Production
**Severity**: Critical
**Location**: `config/application.yml:12`
**Issue**: Admin password set to 'admin'
**Remediation**: 
1. Rotate credentials immediately
2. Use environment variables or secrets manager
3. See: solace-security-review skill

### [RELIABILITY-001] No Reconnection Strategy
**Severity**: Critical
**Location**: `src/SolaceConnection.java:45`
**Issue**: RECONNECT_RETRIES not configured
**Remediation**:
1. Add reconnection configuration
2. Implement session event handlers
3. See: solace-app-review skill

---

## High Priority Findings

### [QUEUE-001] Missing Dead Letter Queue
**Severity**: High
**Location**: Broker configuration - `order-processing` queue
**Issue**: No DMQ configured, failed messages lost after max redelivery
**Remediation**:
1. Create DMQ: `order-processing.dmq`
2. Configure max-redelivery-count: 3
3. Monitor DMQ depth
4. See: solace-queue-review skill

---

## Medium Priority Findings

### [PERF-001] Synchronous Publishing in Hot Path
**Severity**: Medium
**Location**: `src/OrderService.java:78`
**Issue**: Blocking publish call in request handler
**Remediation**:
1. Use async publish with confirmation callback
2. Consider batching
3. See: solace-performance-review skill

---

## Low Priority Findings

### [DESIGN-001] Inconsistent Topic Naming
**Severity**: Low
**Location**: Various
**Issue**: Mix of camelCase and kebab-case in topics
**Remediation**:
1. Standardize on one convention (recommend kebab-case)
2. Document in topic taxonomy
3. See: solace-topic-review skill

---

## Recommendations

### Immediate Actions (Before Production)
1. Fix all Critical findings
2. Address High priority security items
3. Implement basic monitoring

### Short-term Actions (Next Sprint)
1. Address remaining High findings
2. Implement DMQ monitoring
3. Document topic taxonomy

### Long-term Improvements
1. Address Medium findings
2. Performance optimization
3. Enhanced observability

---

## Appendix

### Files Reviewed
- `src/main/java/com/example/*.java`
- `config/application.yml`
- Broker configuration via SEMP

### Tools Used
- Static code analysis
- SEMP API queries
- solace-* review skills

### Assessment Criteria
- Solace Best Practices documentation
- Industry security standards
- Performance benchmarks
```

## Specialized Reviews

This skill can invoke specialized review skills for deeper analysis:

| Aspect | Specialized Skill |
|--------|-------------------|
| Application Code | `solace-app-review` |
| Topic Design | `solace-topic-review` |
| Security Config | `solace-security-review` |
| Queue Config | `solace-queue-review` |
| Performance | `solace-performance-review` |
| Event Portal Design | `solace-event-design-review` |
| SEMP Usage | `solace-semp-review` |

## Scoring Guide

### Security (25 points)

| Item | Points | Criteria |
|------|--------|----------|
| Authentication | 5 | Appropriate scheme, no defaults |
| ACL Profiles | 5 | Least privilege, explicit rules |
| TLS/Encryption | 5 | TLS enabled, valid certs |
| Credential Management | 5 | No hardcoding, rotation policy |
| Audit Logging | 5 | Enabled where required |

### Reliability (20 points)

| Item | Points | Criteria |
|------|--------|----------|
| Reconnection | 5 | HA: 5 min duration (20 retries × 3s × 5/host). Replication: infinite retries (-1) |
| Error Handling | 5 | All errors handled, DLQ configured, max-redelivery-count finite (3-5) |
| HA Configuration | 5 | Host lists for software brokers, proper failover tested |
| Graceful Shutdown | 5 | Clean disconnect, drain, resources closed |

### Performance (20 points)

| Item | Points | Criteria |
|------|--------|----------|
| QoS Selection | 5 | Appropriate for use case |
| Connection Patterns | 5 | Efficient session usage, TCP buffers sized for WAN if needed |
| Message Handling | 5 | No blocking in callbacks, prompt acknowledgment, batch sending where applicable |
| Resource Sizing | 5 | G-1 queue work units < 20,000 per client (formula: flows × window × msg_bytes ÷ 2048) |

### Observability (15 points)

| Item | Points | Criteria |
|------|--------|----------|
| Health Checks | 5 | /health-check/guaranteed-active and /health-check/direct-active exposed |
| Metrics | 5 | Minimum events monitored (see official list below) |
| Alerting | 5 | Thresholds configured for spool >80%, slow subscribers, reconnects |

### Design (15 points)

| Item | Points | Criteria |
|------|--------|----------|
| Topic Architecture | 5 | Hierarchical, consistent |
| Schema Design | 5 | Versioned, documented |
| Patterns | 5 | Appropriate for use case |

### Operations (5 points)

| Item | Points | Criteria |
|------|--------|----------|
| Documentation | 2 | Topics, queues documented |
| Procedures | 3 | Runbooks, on-call |

## Official Solace Best Practice Values

### HA Reconnection Settings (Official)

```yaml
# HA failover typically completes in ~30 seconds
# Configure for 5+ minute reconnect duration

session_properties:
  connect_retries: 1
  reconnect_retries: 20              # -1 for replication (infinite)
  reconnect_retry_wait_ms: 3000
  connect_retries_per_host: 5
  reapply_subscriptions: true

# Host list for software event broker HA
host: "tcp://primary:55555,tcp://backup:55555"
```

### G-1 Queue Sizing (Per-Client Egress Priority Queue)

```markdown
## Work Unit Calculation

Work unit = 2,048 bytes
Default G-1 queue max depth = 20,000 work units (~40 MB)

Formula:
  Total work units = (Num Flows) × (Window Size) × (Avg Msg Bytes) ÷ 2048

Example (PROBLEM):
  10 flows × 255 window × 20KB messages = 25,500 work units ❌ EXCEEDS LIMIT

Example (SOLUTION):
  10 flows × 25 window × 20KB messages = 2,500 work units ✅ OK
```

### AD Window Coordination

```markdown
## AD Window vs max-delivered-unacked-msgs-per-flow

AD window size (client) ≤ max-delivered-unacked-msgs-per-flow (queue)

If AD window > max-unacked:
  - Broker limits delivery to max-unacked anyway
  - API cannot acknowledge messages efficiently
  - Results in slower message delivery

For single-message processing:
  - Set both to 1 for consistent delivery timing
```

### Minimum Recommended Monitoring Events (Official)

```markdown
## System Events
- EVENT_SYSTEM_REDUNDANCY_ACTIVITY_STATE_CHANGED
- EVENT_SYSTEM_REDUNDANCY_MODE_CHANGED
- EVENT_SYSTEM_VIRTUAL_ROUTER_NAME_ASSIGN_CHANGED

## VPN Events
- EVENT_VPN_BRIDGE_UP
- EVENT_VPN_BRIDGE_DOWN
- EVENT_VPN_DMR_CONNECTED
- EVENT_VPN_DMR_DISCONNECTED

## Client Events
- EVENT_CLIENT_CONNECT
- EVENT_CLIENT_DISCONNECT

## Queue Events
- EVENT_QUEUE_BIND
- EVENT_QUEUE_UNBIND
- EVENT_QUEUE_SPOOL_QUOTA_EXCEEDED
```

### Keep-Alive Alignment

```markdown
## Keep-Alive Configuration

Default broker TCP keep-alive:
  - 3s idle before first probe
  - 5 probes at 1s interval
  - Total: 8 seconds to detect failure

Default client API keep-alive:
  - 3s interval
  - 3 missed probes
  - Total: 9 seconds to detect failure

⚠️ WARNING: If client keep-alive is much more aggressive than broker,
connection may drop prematurely!

BAD:  keep_alive_interval_ms: 500  (too aggressive)
GOOD: keep_alive_interval_ms: 3000 (matches broker)
```

### Broker Configuration Checklist

```markdown
## Must-Enable Settings

- [ ] reject-msg-to-sender-on-discard: enabled
      (Unless multi-queue fanout where one queue full shouldn't block others)

- [ ] respect-ttl: enabled (on queues using TTL)

- [ ] max-redelivery-count: 3-10 (finite, not infinite)

- [ ] Dead Message Queue: configured for critical queues
```

## Reference Links

- **Best Practices**: https://docs.solace.com/Best-Practices/Best-Practices.htm
- **Solace Masterclass Codelab**: https://codelabs.solace.dev/codelabs/solace-masterclass/ (198 min)
- **Security Guide**: https://docs.solace.com/Security/Security.htm

## Further Reading

- [Security Checklist](./references/security-checklist.md)
- [Performance Checklist](./references/performance-checklist.md)
- [Production Readiness](./references/production-readiness.md)
