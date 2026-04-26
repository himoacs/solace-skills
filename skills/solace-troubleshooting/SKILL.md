---
name: solace-troubleshooting
description: 'Solace troubleshooting and debugging guidance. Use for connection failures, message delivery issues, performance problems, queue binding errors, client disconnects, spool quota exceeded, and log analysis.'
argument-hint: 'connection, delivery, performance, queue, or logs'
---

# Solace Troubleshooting

Expert guidance for diagnosing and resolving Solace PubSub+ issues.

## When to Use

- **Connection Problems**: Failures, timeouts, disconnects
- **Message Delivery**: Lost or stuck messages
- **Performance Issues**: Latency, throughput problems
- **Queue Issues**: Binding failures, spool full
- **Log Analysis**: Error interpretation

## Quick Diagnostics

### Health Check Commands

```bash
# Broker health
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/about/api" | jq

# Message VPN status
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns" \
  | jq '.data[] | {name: .msgVpnName, operational: .state, spoolUsage: .msgSpoolUsage}'

# Client connections
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/clients" \
  | jq '.data | length'
```

## Connection Issues

### Connection Refused

**Symptoms**: `Connection refused` or `Unable to connect`

**Checklist**:
```markdown
1. [ ] Verify broker is running: `docker ps` or `systemctl status solace`
2. [ ] Check correct port: SMF=55555, TLS=55443, MQTT=1883
3. [ ] Verify VPN exists and is enabled
4. [ ] Check client username is enabled
5. [ ] Verify network connectivity: `telnet broker 55555`
6. [ ] Check firewall rules
```

**Diagnostic Commands**:
```bash
# Check listening ports
docker exec solace netstat -tlnp | grep LISTEN

# Verify service status
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/service" \
  | jq '.data | {smf: .smfPlainTextEnabled, smfTls: .smfTlsEnabled}'

# Check VPN state
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default" \
  | jq '{operationalState: .state, enabled: .enabled}'
```

### Authentication Failures

**Symptoms**: `Authentication failed` or `Not authorized`

```bash
# Verify client username exists and is enabled
curl -u admin:admin "http://localhost:8080/SEMP/v2/config/msgVpns/default/clientUsernames" \
  | jq '.data[] | {name: .clientUsername, enabled: .enabled}'

# Check ACL profile permissions
curl -u admin:admin "http://localhost:8080/SEMP/v2/config/msgVpns/default/aclProfiles" \
  | jq '.data[] | {profile: .aclProfileName, defaultPublish: .publishTopicDefaultAction}'

# Test password (create test session)
curl -X POST "http://localhost:9000/TOPIC/test/topic" \
  -u testuser:testpassword \
  -H "Content-Type: text/plain" \
  -d "test" -v
```

### TLS/SSL Errors

**Symptoms**: `SSL handshake failed` or `Certificate validation failed`

```bash
# Test TLS connection
openssl s_client -connect broker:55443 -showcerts

# Verify certificate dates
openssl s_client -connect broker:55443 2>/dev/null | openssl x509 -noout -dates

# Check hostname match
openssl s_client -connect broker:55443 2>/dev/null | openssl x509 -noout -subject
```

**Common Fixes**:
```java
// Disable hostname verification (development only!)
sessionProps.setProperty(SessionProperties.SSL_VALIDATE_CERTIFICATE_HOST, false);

// Use custom trust store
sessionProps.setProperty(SessionProperties.SSL_TRUST_STORE, "/path/to/truststore.jks");
```

## Message Delivery Issues

### Messages Not Delivered

**Symptoms**: Publisher sends but subscriber doesn't receive

**Diagnostic Steps**:
```bash
# 1. Check subscription exists
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/subscribers" \
  | jq '.data[] | {topic: .topicEndpointName}'

# 2. Check if messages are being published
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/clients" \
  | jq '.data[] | {client: .clientName, publishedMsgs: .dataRxMsgCount}'

# 3. Check ACL blocks
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/clients" \
  | jq '.data[] | {client: .clientName, aclDeniedPublish: .aclProfilePublishTopicDenyCount}'

# 4. Verify topic pattern match
# Subscriber: acme/orders/*
# Publisher:  acme/orders/created/v1  <- Won't match! Use > not *
```

### Messages Stuck in Queue

**Symptoms**: Queue depth growing, consumers not receiving

```bash
# Check queue consumers
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues/my-queue" \
  | jq '{
    msgCount: .data.msgCount,
    bindCount: .data.bindCount,
    consumerCount: .data.consumerCount,
    egressEnabled: .data.egressEnabled
  }'

# Check flow state
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues/my-queue/msgs" \
  | jq '.data[0:5]'  # First 5 messages
```

**Common Causes**:
- Consumer not acknowledging messages
- Consumer disconnected
- Egress disabled on queue
- Message selector not matching

### Duplicate Messages

**Symptoms**: Same message received multiple times

```bash
# Check redelivery count
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues/my-queue" \
  | jq '{redeliveredMsgCount: .data.redeliveredMsgCount}'
```

**Root Causes**:
- Consumer crashing before ack
- Network timeout causing retry
- HA failover during processing

**Solution**: Implement idempotent message handling with message ID deduplication.

## Queue Issues

### Queue Bind Failure

**Symptoms**: `Queue unavailable` or `Bind failed`

```bash
# Check queue exists and is operational
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues" \
  | jq '.data[] | {name: .queueName, state: .state}'

# Check exclusive access
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues/my-queue" \
  | jq '{accessType: .data.accessType, bindCount: .data.bindCount}'

# If exclusive queue, only one consumer allowed
```

### Spool Quota Exceeded

**Symptoms**: `Spool quota exceeded` or `Queue full`

```bash
# Check spool usage
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default" \
  | jq '{spoolUsage: .data.msgSpoolUsage, maxSpool: .data.maxMsgSpoolUsage}'

# Check individual queue
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues/my-queue" \
  | jq '{spoolUsage: .data.msgSpoolUsage, maxSpool: .data.maxMsgSpoolUsage}'
```

**Solutions**:
```bash
# Increase queue spool
curl -X PATCH "http://localhost:8080/SEMP/v2/config/msgVpns/default/queues/my-queue" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"maxMsgSpoolUsage": 5000}'

# Increase VPN spool
curl -X PATCH "http://localhost:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"maxMsgSpoolUsage": 50000}'

# Clear stuck messages (caution!)
curl -X PUT "http://localhost:8080/SEMP/v2/action/msgVpns/default/queues/my-queue/clearMsgs" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{}'
```

## Performance Issues

### High Latency

**Symptoms**: Slow message delivery

**Diagnostics**:
```bash
# Check broker load
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/about/api" | jq

# Check client stats
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/clients" \
  | jq '.data[] | {
    client: .clientName,
    avgRxRate: .averageRxMsgRate,
    avgTxRate: .averageTxMsgRate
  }'

# Check queue depth (backlog indicates slow consumer)
curl -u admin:admin "http://localhost:8080/SEMP/v2/monitor/msgVpns/default/queues" \
  | jq '.data[] | {queue: .queueName, depth: .msgCount}'
```

**Tuning Options**:
```java
// Client-side optimizations
sessionProps.setProperty(SessionProperties.PUB_WINDOW_SIZE, "100");
sessionProps.setProperty(SessionProperties.SEND_BUFFER_SIZE, "65536");
sessionProps.setProperty(SessionProperties.TCP_NODELAY, "true");
```

### Low Throughput

**Symptoms**: Message rate below expected

```bash
# Check message rates
watch -n 1 'curl -s -u admin:admin \
  "http://localhost:8080/SEMP/v2/monitor/msgVpns/default" \
  | jq "{rxRate: .data.rxMsgRate, txRate: .data.txMsgRate}"'
```

**Bottleneck Checklist**:
```markdown
- [ ] Check client-side batching
- [ ] Verify network bandwidth
- [ ] Check broker CPU/memory
- [ ] Review message size
- [ ] Check if using persistent (slower than direct)
- [ ] Verify consumer acknowledgment rate
```

## Log Analysis

### Important Log Files

```bash
# Docker logs
docker logs solace 2>&1 | grep -i error

# Key log locations (inside container)
docker exec solace ls /var/solace/logs/
# - event.log: Client and system events
# - command.log: CLI/SEMP commands
# - system.log: System-level events
```

### Common Error Patterns

```bash
# Authentication failures
grep -i "authentication" /var/solace/logs/event.log

# Connection issues
grep -i "connect\|disconnect" /var/solace/logs/event.log

# Quota exceeded
grep -i "quota\|spool" /var/solace/logs/event.log

# SSL/TLS errors
grep -i "ssl\|tls\|certificate" /var/solace/logs/event.log
```

### Error Code Reference

| Code | Meaning | Action |
|------|---------|--------|
| `SOLCLIENT_FAIL` | General failure | Check logs for details |
| `SOLCLIENT_WOULD_BLOCK` | Flow control | Implement backpressure |
| `SOLCLIENT_NOT_READY` | Session not connected | Wait for reconnection |
| `SOLCLIENT_EOS` | End of stream | Normal, consumer stopped |
| `SOLCLIENT_TIMEOUT` | Operation timed out | Increase timeout or retry |

## Debugging Tools

### Client Debug Logging

```java
// Enable verbose logging
SolaceFactory.setLogLevel(SolaceFactory.LOG_DEBUG);
```

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Capture Network Traffic

```bash
# Capture Solace traffic
tcpdump -i any port 55555 -w solace.pcap

# Analyze with Wireshark
wireshark solace.pcap
```

## Reference Links

- **Troubleshooting Guide**: https://docs.solace.com/Troubleshooting/Troubleshooting-Overview.htm
- **Error Codes**: https://docs.solace.com/API-Developer-Online-Ref-Documentation/java/index.html
- **SEMP Monitoring**: https://docs.solace.com/SEMP/SEMP-Get-Started.htm

## Further Reading

- [Error Code Reference](./references/error-codes.md)
- [Performance Tuning](./references/performance-tuning.md)
- [Log Analysis](./references/log-analysis.md)
