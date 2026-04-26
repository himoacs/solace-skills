---
name: solace-cloud-platform
description: 'Solace Cloud Platform guidance for managed broker services. Use for Cloud Console navigation, Mission Control broker management, Solace Insights monitoring, service provisioning, Cloud REST APIs, multi-cloud deployment, and cluster management.'
argument-hint: 'console, mission-control, insights, provisioning, or api'
---

# Solace Cloud Platform

Expert guidance for Solace Cloud - a fully managed event streaming platform providing event broker services, monitoring, and management.

## When to Use

- **Service Management**: Provisioning and managing event broker services
- **Cloud Console**: Navigating the Solace Cloud Console
- **Mission Control**: Advanced broker management and monitoring
- **Insights**: Viewing metrics, alerts, and operational status
- **Cloud APIs**: Automating cloud operations via REST APIs
- **Multi-Cloud**: Deploying across AWS, Azure, GCP

## Core Components

### Solace Cloud Console

| Section | Purpose |
|---------|---------|
| **Cluster Manager** | Create and manage event broker services |
| **Mission Control** | Advanced broker configuration and monitoring |
| **Event Portal** | EDA design and governance |
| **Insights** | Operational dashboards and alerts |
| **Micro-Integrations** | Managed integration connectors |
| **Token Management** | API tokens for automation |

### Service Tiers

| Tier | Use Case | Features |
|------|----------|----------|
| **Developer** | Development/testing | Free, limited connections |
| **Enterprise** | Production workloads | HA, guaranteed messaging |
| **Premier** | Mission-critical | Maximum throughput, support SLA |

## Service Provisioning

### Creating an Event Broker Service

1. Navigate to **Cluster Manager** → **Create Service**
2. Configure:
   - **Name**: `prod-orders-broker`
   - **Cloud Provider**: AWS / Azure / GCP
   - **Region**: us-east-1
   - **Service Type**: Enterprise
   - **Messaging Capacity**: Based on requirements

### Service Configuration

```yaml
# Service settings
name: prod-orders-broker
serviceType: enterprise

# Deployment
cloud: aws
region: us-east-1
availabilityZone: multi-az  # for HA

# Messaging capacity
messagingStorage: 250  # GB
maxConnections: 1000
maxThroughput: 1000  # msgs/sec

# Features
eventMeshEnabled: true
haEnabled: true
clientCertificateAuthEnabled: true
```

### Connection Details

After provisioning, get connection details:

```bash
# Cloud Console → Service → Connect tab

# SMF Connection
Host: mr-connection-xxx.messaging.solace.cloud
Port: 55555 (TCP) / 55443 (TLS)
Message VPN: <service-name>
Client Username: solace-cloud-client

# WebSocket
ws://mr-connection-xxx.messaging.solace.cloud:80
wss://mr-connection-xxx.messaging.solace.cloud:443

# REST
https://mr-connection-xxx.messaging.solace.cloud:9443
```

## Mission Control

### VPN Configuration

```bash
# Access via Cloud Console → Service → Manage → Mission Control

# Or via SEMP API
curl -X GET \
  "https://${SERVICE_URL}:943/SEMP/v2/config/msgVpns/${VPN_NAME}" \
  -u "${ADMIN_USER}:${ADMIN_PASSWORD}"
```

### Queue Management

```bash
# Create queue via Mission Control SEMP
curl -X POST \
  "https://${SERVICE_URL}:943/SEMP/v2/config/msgVpns/${VPN_NAME}/queues" \
  -u "${ADMIN_USER}:${ADMIN_PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "order-processing",
    "accessType": "exclusive",
    "permission": "consume",
    "ingressEnabled": true,
    "egressEnabled": true,
    "maxMsgSpoolUsage": 5000
  }'
```

### Client Profile Configuration

```json
{
  "clientProfileName": "production-apps",
  "allowGuaranteedMsgSendEnabled": true,
  "allowGuaranteedMsgReceiveEnabled": true,
  "maxConnectionCount": 100,
  "maxEgressFlowCount": 1000,
  "maxIngressFlowCount": 1000,
  "maxSubscriptionCount": 500000
}
```

## Solace Cloud REST APIs

### Authentication

```bash
# Create API token: Cloud Console → Token Management → Create Token

# Use token in requests
export SOLACE_CLOUD_TOKEN="your-api-token"

# Base URL
CLOUD_API="https://api.solace.cloud/api/v2"
```

### Service Management API

```bash
# List all services
curl -X GET "${CLOUD_API}/missionControl/services" \
  -H "Authorization: Bearer ${SOLACE_CLOUD_TOKEN}"

# Get service details
curl -X GET "${CLOUD_API}/missionControl/services/${SERVICE_ID}" \
  -H "Authorization: Bearer ${SOLACE_CLOUD_TOKEN}"

# Create service
curl -X POST "${CLOUD_API}/missionControl/services" \
  -H "Authorization: Bearer ${SOLACE_CLOUD_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "new-broker-service",
    "serviceTypeId": "enterprise",
    "serviceClassId": "enterprise-250-nano",
    "datacenterId": "aws-us-east-1a"
  }'
```

### Datacenter Discovery

```bash
# List available datacenters
curl -X GET "${CLOUD_API}/missionControl/datacenters" \
  -H "Authorization: Bearer ${SOLACE_CLOUD_TOKEN}"

# Response example
{
  "data": [
    {
      "id": "aws-us-east-1a",
      "name": "AWS US East (N. Virginia)",
      "provider": "aws",
      "regionId": "us-east-1"
    },
    {
      "id": "azure-eastus",
      "name": "Azure East US",
      "provider": "azure"
    }
  ]
}
```

## Solace Insights

### Dashboard Overview

| Dashboard | Metrics |
|-----------|---------|
| **Service Health** | Connection status, message rates, errors |
| **Capacity** | Spool usage, connection counts, queue depths |
| **Performance** | Latency percentiles, throughput |
| **Events** | Alerts, anomalies, threshold breaches |

### Configuring Alerts

```bash
# Via Cloud API
curl -X POST "${CLOUD_API}/insights/alerts" \
  -H "Authorization: Bearer ${SOLACE_CLOUD_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "High Queue Depth Alert",
    "serviceId": "service-123",
    "metric": "queue.messageCount",
    "queueName": "order-processing",
    "threshold": 10000,
    "operator": "greaterThan",
    "severity": "warning",
    "notification": {
      "type": "email",
      "recipients": ["ops@example.com"]
    }
  }'
```

### Key Metrics to Monitor

| Metric | Warning Threshold | Critical Threshold |
|--------|-------------------|-------------------|
| Spool Usage % | 70% | 90% |
| Connection Count | 80% of max | 95% of max |
| Queue Depth | Depends on SLA | Depends on SLA |
| Message Rate | Baseline + 50% | Baseline + 100% |
| Client Reconnects | 10/min | 50/min |

## Multi-Cloud Deployment

### Event Mesh Configuration

```yaml
# Connect brokers across clouds via DMR
primaryService:
  id: aws-us-east-service
  cloud: aws
  region: us-east-1
  
peerServices:
  - id: azure-westus-service
    cloud: azure
    region: westus2
    
  - id: gcp-us-central-service
    cloud: gcp
    region: us-central1

# Enable DMR cluster links
dmr:
  clusterPassword: ${DMR_PASSWORD}
  authenticationMode: internal
```

### Cross-Cloud Topic Routing

```yaml
# Topics are automatically routed across the mesh
# when services are connected via DMR

# Subscribers in any region receive:
# - acme/orders/> (all order events)
# - acme/inventory/> (all inventory events)

# No additional configuration needed for topic routing
```

## Best Practices

### Service Naming

```
<environment>-<application>-<region>
Examples:
- prod-orders-useast
- dev-payments-euwest
- staging-inventory-apac
```

### Security Configuration

1. **Client Certificates**: Enable for production
2. **OAuth**: Use for application authentication
3. **IP Filtering**: Restrict access by IP ranges
4. **Audit Logging**: Enable for compliance

### Cost Optimization

1. **Right-size services**: Match tier to actual usage
2. **Use Developer tier**: For non-production
3. **Monitor spool usage**: Avoid over-provisioning storage
4. **Clean up unused services**: Regular audits

## Reference Links

- **Solace Cloud Documentation**: https://docs.solace.com/Cloud/cloud-lp.htm
- **Cloud REST API Reference**: https://api.solace.dev/cloud/reference
- **Mission Control Guide**: https://docs.solace.com/Cloud/ght_set_up_service.htm
- **Insights Documentation**: https://docs.solace.com/Cloud/Insights/Insights.htm
- **Pricing Calculator**: https://solace.com/cloud-pricing/

## Further Reading

- [API Token Management](./references/api-tokens.md)
- [Service Provisioning](./references/provisioning.md)
- [Insights Alerting](./references/alerts.md)
- [Multi-Cloud Architecture](./references/multi-cloud.md)
