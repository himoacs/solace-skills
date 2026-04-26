---
name: solace-operations
description: 'Solace broker operations guidance. Use for SEMP v2 API administration, CLI commands, monitoring endpoints, queue management, client management, high availability configuration, and capacity planning.'
argument-hint: 'semp, cli, monitoring, ha, or scaling'
---

# Solace Operations

Expert guidance for operating and administering Solace PubSub+ brokers.

## When to Use

- **Broker Administration**: SEMP API and CLI operations
- **Monitoring**: Metrics, alerts, and health checks
- **Capacity Planning**: Sizing and scaling
- **High Availability**: HA configuration and failover
- **Troubleshooting**: Operational issues

## SEMP v2 API

### Authentication

```bash
# Basic Authentication
curl -u admin:admin "http://broker:8080/SEMP/v2/config/msgVpns"

# OAuth Bearer Token
curl -H "Authorization: Bearer ${TOKEN}" "http://broker:8080/SEMP/v2/config/msgVpns"
```

### Common Operations

#### List VPNs

```bash
curl -u admin:admin "http://broker:8080/SEMP/v2/config/msgVpns" | jq '.data[].msgVpnName'
```

#### Create Queue

```bash
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/queues" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "order-processing",
    "accessType": "exclusive",
    "egressEnabled": true,
    "ingressEnabled": true,
    "maxMsgSpoolUsage": 5000,
    "permission": "consume",
    "respectTtlEnabled": true
  }'
```

#### Add Queue Subscription

```bash
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/queues/order-processing/subscriptions" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "subscriptionTopic": "acme/orders/>"
  }'
```

#### Create Client Username

```bash
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/clientUsernames" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "clientUsername": "order-service",
    "enabled": true,
    "password": "secure-password",
    "aclProfileName": "order-service-acl",
    "clientProfileName": "default"
  }'
```

### SEMP Pagination

```bash
# Get first page
response=$(curl -s -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/msgVpns/default/queues?count=100")

# Check for more pages
next_uri=$(echo $response | jq -r '.meta.paging.nextPageUri // empty')

while [ -n "$next_uri" ]; do
  response=$(curl -s -u admin:admin "$next_uri")
  # Process response
  next_uri=$(echo $response | jq -r '.meta.paging.nextPageUri // empty')
done
```

## CLI Operations

### Basic Commands

```bash
# Enter CLI
docker exec -it solace solacectl

# Show message VPN
solacectl show message-vpn default

# Show queues
solacectl show queue * message-vpn default

# Show clients
solacectl show client * message-vpn default

# Show queue stats
solacectl show queue order-processing message-vpn default detail
```

### Queue Management

```bash
# Clear queue messages
solacectl admin queue order-processing message-vpn default clear-messages

# Shutdown queue
solacectl admin queue order-processing message-vpn default shutdown ingress

# Enable queue
solacectl admin queue order-processing message-vpn default no shutdown ingress

# Delete queue
solacectl admin queue order-processing message-vpn default force-delete
```

### Client Management

```bash
# Disconnect specific client
solacectl admin client-username order-service message-vpn default disconnect

# Clear client stats
solacectl admin client * message-vpn default clear-stats
```

## Monitoring

### Health Check Endpoints

```bash
# Broker health
curl "http://broker:5550/health-check/guaranteed-active"

# Direct messaging health
curl "http://broker:5550/health-check/direct-active"

# Response codes:
# 200 - Healthy
# 503 - Unhealthy
```

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Queue spool usage | Messages in queue | > 80% max |
| Connection count | Active clients | > 80% max |
| Message rate | Msgs/sec | Anomaly detection |
| Discard rate | Dropped messages | > 0 |
| Client reconnects | Session restarts | Spike detection |

### SEMP Monitor Queries

```bash
# Queue depth and spool
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/msgVpns/default/queues" | \
  jq '.data[] | {name: .queueName, msgs: .msgCount, spoolMB: (.msgSpoolUsage/1048576)}'

# Client connections
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/msgVpns/default/clients" | \
  jq '.data[] | {client: .clientName, user: .clientUsername, uptime: .uptime}'

# Message VPN stats
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/msgVpns/default" | \
  jq '{
    connections: .data.connectionCount,
    msgRateIn: .data.msgRateInMsgsPerSecond,
    msgRateOut: .data.msgRateOutMsgsPerSecond,
    spoolUsageMB: (.data.msgSpoolUsage/1048576)
  }'
```

### Prometheus Metrics

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'solace'
    static_configs:
      - targets: ['broker:9628']
    metrics_path: /solace

# Key metrics:
# solace_queue_messages_spooled
# solace_vpn_connections
# solace_client_slow_subscriber
# solace_queue_bind_count
```

## High Availability

### HA Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| Active/Standby | Hot standby | General HA |
| Replication | Multi-site async | DR |
| DMR | Dynamic Message Routing | Event mesh |

### HA Configuration

```bash
# Check HA status
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/msgVpns/default/bridges" | \
  jq '.data[] | {name: .bridgeName, state: .state, lastFailureReason}'

# Redundancy status
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/monitor/about/api" | \
  jq '{
    role: .data.sempVersion,
    isActive: .data.isActive
  }'
```

### Failover Testing

```bash
# Trigger admin shutdown (controlled failover)
curl -X PATCH "http://primary:8080/SEMP/v2/config" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"systemShutdown": "reboot"}'

# Monitor failover
watch -n 1 'curl -s http://backup:5550/health-check/guaranteed-active'
```

## Official HA/Redundancy Best Practices (from Solace Documentation)

### Client Reconnection Settings for HA

```markdown
## HA Failover Timing

HA failovers typically complete within 30 seconds.
Configure client reconnection duration for at least 5 minutes (300 seconds).
```

```java
// Java JCSMP - Recommended HA reconnect settings
JCSMPProperties properties = new JCSMPProperties();
properties.setProperty(JCSMPProperties.CONNECT_RETRIES, 1);
properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, 20);
properties.setProperty(JCSMPProperties.RECONNECT_RETRY_WAIT_IN_MILLIS, 3000);
properties.setProperty(JCSMPProperties.CONNECT_RETRIES_PER_HOST, 5);

// Total reconnect duration: 20 × 3s × 5 = 300 seconds (5 minutes)
```

```c
// C API - Recommended HA reconnect settings
const char *sessionProps[] = {
    SOLCLIENT_SESSION_PROP_CONNECT_RETRIES, "1",
    SOLCLIENT_SESSION_PROP_RECONNECT_RETRIES, "20",
    SOLCLIENT_SESSION_PROP_RECONNECT_RETRY_WAIT_MS, "3000",
    SOLCLIENT_SESSION_PROP_CONNECT_RETRIES_PER_HOST, "5",
    NULL
};
```

### Host Lists for Software Event Brokers

```yaml
# Software event brokers use host lists for HA (not IP takeover)
# Clients should configure both primary and backup addresses

host_list: "tcp://primary:55555,tcp://backup:55555"

# Order matters: Primary first, backup second
# Set connect_retries_per_host low (1) for quick failover between hosts
```

```java
// Java - Host list configuration
JCSMPProperties properties = new JCSMPProperties();
properties.setProperty(JCSMPProperties.HOST, "tcp://primary:55555,tcp://backup:55555");
properties.setProperty(JCSMPProperties.CONNECT_RETRIES_PER_HOST, 1);  // Quick failover
```

### Replication Failover Configuration

```markdown
## Replication Failover Characteristics

- Duration is NON-DETERMINISTIC (may require operator intervention)
- Can take minutes to hours
- Client should retry indefinitely

**Recommendation:** Set reconnect_retries to -1 (infinite)
```

```java
// Java - Replication-aware clients
properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, -1);  // Infinite

// Host list should contain BOTH sites
properties.setProperty(JCSMPProperties.HOST, 
    "tcp://site-a-primary:55555,tcp://site-a-backup:55555," +
    "tcp://site-b-primary:55555,tcp://site-b-backup:55555");
```

### Replication Failover Host List Rules

```markdown
## Important: DO NOT use host lists in Active/Active Replication

Host lists are for:
✅ HA redundant groups (software event broker)
✅ Active/Standby replication failover

Host lists must NOT be used when:
❌ Active/Active replication where clients consume from BOTH sites
```

### Client Session Re-Establishment (Pre-7.1.2)

```markdown
## Legacy API Considerations

For API versions < 7.1.2:
- Sessions need re-establishment after replication failover
- Flow state is NOT synchronized from old active to new active
- Catch `unknown_flow_name` event and re-establish session

For API versions >= 7.1.2:
- API is replication-aware
- Session re-establishment is handled transparently
```

### Monitoring Events for HA

```bash
# Minimum recommended events for management applications to monitor
# (from official Solace documentation)

# System events
EVENT_SYSTEM_REDUNDANCY_ACTIVITY_STATE_CHANGED
EVENT_SYSTEM_REDUNDANCY_MODE_CHANGED
EVENT_SYSTEM_VIRTUAL_ROUTER_NAME_ASSIGN_CHANGED

# VPN events  
EVENT_VPN_BRIDGE_UP
EVENT_VPN_BRIDGE_DOWN
EVENT_VPN_DMR_CONNECTED
EVENT_VPN_DMR_DISCONNECTED

# Client events
EVENT_CLIENT_CONNECT
EVENT_CLIENT_DISCONNECT

# Queue events
EVENT_QUEUE_BIND
EVENT_QUEUE_UNBIND
EVENT_QUEUE_SPOOL_QUOTA_EXCEEDED
```

### HA Health Check Script

```bash
#!/bin/bash
# Comprehensive HA health check

BROKER_PRIMARY="broker-primary:8080"
BROKER_BACKUP="broker-backup:8080"
AUTH="admin:admin"

echo "=== HA Health Check ==="

# Check primary
echo -n "Primary Status: "
curl -s -u $AUTH "http://$BROKER_PRIMARY/SEMP/v2/monitor" | \
  jq -r '.data.guaranteedMsgingEnabled // "NOT_REACHABLE"'

# Check backup
echo -n "Backup Status:  "
curl -s -u $AUTH "http://$BROKER_BACKUP/SEMP/v2/monitor" | \
  jq -r '.data.guaranteedMsgingEnabled // "NOT_REACHABLE"'

# Check active node
for broker in $BROKER_PRIMARY $BROKER_BACKUP; do
  active=$(curl -s "http://$broker:5550/health-check/guaranteed-active" -o /dev/null -w "%{http_code}")
  if [ "$active" == "200" ]; then
    echo "Active Node:    $broker"
    break
  fi
done

# Check replication bridges if configured
echo -e "\n=== Replication Bridges ==="
curl -s -u $AUTH "http://$BROKER_PRIMARY/SEMP/v2/monitor/msgVpns/default/bridges" | \
  jq '.data[] | {bridge: .bridgeName, state: .inboundState, lastFailure: .lastFailureReason}'
```

## Capacity Planning

### Message VPN Limits

```bash
# Get current limits
curl -u admin:admin \
  "http://broker:8080/SEMP/v2/config/msgVpns/default" | \
  jq '{
    maxConnections: .data.maxConnectionCount,
    maxSubscriptions: .data.maxSubscriptionCount,
    maxQueues: .data.maxQueueCount,
    maxSpoolMB: (.data.maxMsgSpoolUsage)
  }'

# Update limits
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "maxConnectionCount": 10000,
    "maxSubscriptionCount": 500000,
    "maxMsgSpoolUsage": 50000
  }'
```

### Sizing Guidelines

| Workload | Connections | Queues | Spool (GB) |
|----------|-------------|--------|------------|
| Development | 100 | 50 | 1 |
| Small | 1,000 | 500 | 10 |
| Medium | 10,000 | 2,000 | 100 |
| Large | 50,000 | 10,000 | 500 |

## Automation Scripts

### Queue Provisioning Script

```python
#!/usr/bin/env python3
import requests
from requests.auth import HTTPBasicAuth

class SolaceAdmin:
    def __init__(self, host, username, password):
        self.base_url = f"http://{host}:8080/SEMP/v2/config"
        self.auth = HTTPBasicAuth(username, password)
    
    def create_queue(self, vpn, queue_name, max_spool_mb=1000, 
                     access_type="exclusive", subscriptions=None):
        url = f"{self.base_url}/msgVpns/{vpn}/queues"
        
        queue_config = {
            "queueName": queue_name,
            "accessType": access_type,
            "egressEnabled": True,
            "ingressEnabled": True,
            "maxMsgSpoolUsage": max_spool_mb,
            "permission": "consume"
        }
        
        response = requests.post(url, json=queue_config, auth=self.auth)
        response.raise_for_status()
        
        # Add subscriptions
        if subscriptions:
            for topic in subscriptions:
                self.add_subscription(vpn, queue_name, topic)
        
        return response.json()
    
    def add_subscription(self, vpn, queue_name, topic):
        url = f"{self.base_url}/msgVpns/{vpn}/queues/{queue_name}/subscriptions"
        response = requests.post(url, json={"subscriptionTopic": topic}, auth=self.auth)
        return response.json()
    
    def delete_queue(self, vpn, queue_name):
        url = f"{self.base_url}/msgVpns/{vpn}/queues/{queue_name}"
        response = requests.delete(url, auth=self.auth)
        return response.ok

# Usage
admin = SolaceAdmin("broker.example.com", "admin", "admin")
admin.create_queue(
    vpn="default",
    queue_name="order-events",
    max_spool_mb=5000,
    subscriptions=["acme/orders/>", "acme/payments/>"]
)
```

### Bulk Operations

```bash
#!/bin/bash
# Create multiple queues from config file

while IFS=, read -r queue_name max_spool topics; do
  curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/queues" \
    -u admin:admin \
    -H "Content-Type: application/json" \
    -d "{
      \"queueName\": \"$queue_name\",
      \"maxMsgSpoolUsage\": $max_spool,
      \"accessType\": \"exclusive\",
      \"egressEnabled\": true,
      \"ingressEnabled\": true
    }"
  
  # Add subscriptions
  for topic in $(echo $topics | tr '|' ' '); do
    curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/queues/$queue_name/subscriptions" \
      -u admin:admin \
      -H "Content-Type: application/json" \
      -d "{\"subscriptionTopic\": \"$topic\"}"
  done
done < queues.csv
```

## Reference Links

- **SEMP v2 API**: https://docs.solace.com/SEMP/Using-SEMP.htm
- **CLI Reference**: https://docs.solace.com/Admin/Solace-CLI/CLI-Quick-Ref.htm
- **Monitoring**: https://docs.solace.com/System-and-Software-Maintenance/Monitoring-Events.htm
- **High Availability**: https://docs.solace.com/Overviews/HA-Overview.htm

## Further Reading

- [SEMP Examples](./references/semp-examples.md)
- [Alerting Configuration](./references/alerting.md)
- [Capacity Planning](./references/capacity.md)
