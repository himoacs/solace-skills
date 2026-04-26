---
name: solace-event-mesh
description: 'Solace Event Mesh (DMR) guidance. Use for Dynamic Message Routing setup, multi-site event distribution, global message routing, DMR cluster configuration, link provisioning, and hybrid cloud connectivity.'
argument-hint: 'dmr-cluster, multi-site, global-routing, or hybrid-cloud'
---

# Solace Event Mesh

Expert guidance for building event meshes with Solace Dynamic Message Routing (DMR).

## When to Use

- **Multi-Site Distribution**: Connect geographically distributed brokers
- **Hybrid Cloud**: Bridge on-premise and cloud brokers
- **Global Routing**: Intelligent cross-region message routing
- **Event Mesh Architecture**: Enterprise-wide event backplane

## Event Mesh Concepts

### Architecture Overview

```
                          ┌─────────────────┐
                          │   Event Mesh    │
                          │     (DMR)       │
    ┌─────────────────────┼─────────────────┼─────────────────────┐
    │                     │                 │                     │
┌───┴───┐             ┌───┴───┐         ┌───┴───┐             ┌───┴───┐
│ Site  │◄───DMR────▶│ Site  │◄──DMR──▶│ Site  │◄───DMR────▶│ Site  │
│  US   │   Link     │  EU   │  Link   │ APAC  │   Link     │ Cloud │
└───────┘             └───────┘         └───────┘             └───────┘
    │                     │                 │                     │
  Apps                  Apps              Apps                  Apps
```

### DMR Benefits

- **Dynamic Routing**: Only delivers messages where needed
- **Subscription Propagation**: Cross-site subscription awareness
- **Protocol Bridging**: Connect SMF, MQTT, AMQP, REST
- **Resilient**: Automatic failover and recovery

## DMR Cluster Configuration

### Create DMR Cluster

```bash
# On each broker, create the cluster
curl -X POST "$SEMP_URL/config/dmrClusters" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "dmrClusterName": "global-mesh",
    "enabled": true,
    "authenticationBasicEnabled": true,
    "authenticationBasicPassword": "cluster-secret",
    "tlsServerCertEnforceTrustedCommonNameEnabled": true
  }'

# Configure local node
curl -X PATCH "$SEMP_URL/config/dmrClusters/global-mesh" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "nodeName": "us-east-broker",
    "enabled": true
  }'
```

### Create External Links

```bash
# Link from US to EU broker
curl -X POST "$SEMP_URL/config/dmrClusters/global-mesh/links" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "remoteNodeName": "eu-west-broker",
    "enabled": true,
    "span": "external",
    "initiator": "local",
    "authenticationBasicPassword": "cluster-secret"
  }'

# Configure remote address
curl -X POST "$SEMP_URL/config/dmrClusters/global-mesh/links/eu-west-broker/remoteAddresses" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "remoteAddress": "tcps://eu-broker.example.com:55443"
  }'
```

### Link Configuration Options

```bash
# Configure link properties
curl -X PATCH "$SEMP_URL/config/dmrClusters/global-mesh/links/eu-west-broker" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "transportCompressedEnabled": true,
    "connectionRetryCount": 3,
    "connectionRetryDelay": 3,
    "queueDeadMsgQueue": "#dmr-dlq"
  }'
```

## Multi-Site Patterns

### Active-Active

```
Site A                          Site B
┌───────────────────┐          ┌───────────────────┐
│ Publishers (50%)  │─┬──────┬─│ Publishers (50%)  │
│ Subscribers (all) │ │ DMR  │ │ Subscribers (all) │
└───────────────────┘ │ Link │ └───────────────────┘
                      └──────┘

# Both sites handle traffic, DMR syncs subscriptions
```

Configuration:
```bash
# Enable subscription export on both sites
curl -X PATCH "$SEMP_URL/config/msgVpns/production" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "dmrEnabled": true
  }'
```

### Active-Passive (DR)

```
Primary Site                    DR Site
┌───────────────────┐          ┌───────────────────┐
│ Publishers        │───────▶│ Passive Replicas  │
│ Subscribers       │   DMR   │ (standby)         │
└───────────────────┘          └───────────────────┘

# DR receives replicated data, activates on failover
```

### Hub-and-Spoke

```
              ┌─────────┐
              │   Hub   │
              │(Central)│
              └────┬────┘
        ┌──────────┼──────────┐
        │          │          │
   ┌────┴────┐┌────┴────┐┌────┴────┐
   │ Spoke 1 ││ Spoke 2 ││ Spoke 3 │
   │ (Site A)││ (Site B)││ (Site C)│
   └─────────┘└─────────┘└─────────┘

# All traffic routes through hub
```

## Global Routing

### Topic-Based Routing

```bash
# Create queue with topic subscription
# Subscriptions propagate across DMR mesh

# Site US subscribes to: orders/US/>
# Site EU subscribes to: orders/EU/>
# Global analytics subscribes to: orders/>

# Publisher in US sends to: orders/US/created
# → Delivered to US local queue immediately
# → Routed via DMR to EU and Global (if subscribed)
```

### No Local Delivery Prevention

```bash
# Prevent local echo for distributed applications
curl -X PATCH "$SEMP_URL/config/msgVpns/production/topicEndpoints" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "noLocalDeliveryEnabled": true
  }'
```

## Hybrid Cloud Configuration

### On-Premise to Solace Cloud

```bash
# On-premise broker: Create link to Solace Cloud
curl -X POST "$SEMP_URL/config/dmrClusters/hybrid-mesh/links" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "remoteNodeName": "solace-cloud-service",
    "enabled": true,
    "span": "external",
    "initiator": "local",
    "authenticationScheme": "basic",
    "authenticationBasicPassword": "cloud-service-password"
  }'

# Add Solace Cloud connection address
curl -X POST "$SEMP_URL/config/dmrClusters/hybrid-mesh/links/solace-cloud-service/remoteAddresses" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "remoteAddress": "tcps://mr-connection-xxx.messaging.solace.cloud:55443"
  }'
```

### Cloud-to-Cloud

```bash
# Connect AWS region to Azure region via DMR
# Each cloud service joins the same DMR cluster

# From AWS broker
curl -X POST "$SEMP_URL/config/dmrClusters/multi-cloud/links" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "remoteNodeName": "azure-westeurope",
    "span": "external"
  }'
```

## Monitoring DMR

### Cluster Status

```bash
# Check cluster health
curl -u admin:admin "$SEMP_URL/monitor/dmrClusters" \
  | jq '.data[] | {
    name: .dmrClusterName,
    up: .up,
    nodeCount: .upstreamConnectionCount
  }'

# Check link status
curl -u admin:admin "$SEMP_URL/monitor/dmrClusters/global-mesh/links" \
  | jq '.data[] | {
    node: .remoteNodeName,
    up: .up,
    established: .bridgeAdminState
  }'
```

### Troubleshooting Links

```bash
# Check connection failures
curl -u admin:admin "$SEMP_URL/monitor/dmrClusters/global-mesh/links" \
  | jq '.data[] | {
    node: .remoteNodeName,
    failureReason: .connectionEstablishmentFailureReason,
    lastFailure: .lastConnectionFailureTime
  }'

# Common issues:
# - Authentication mismatch
# - TLS certificate issues
# - Network connectivity
# - Port blocked by firewall
```

## Best Practices

### Network Planning

```markdown
## DMR Network Checklist

- [ ] Dedicated network path between sites
- [ ] Minimum 10 Mbps bandwidth per link
- [ ] Maximum 150ms RTT latency recommended
- [ ] Firewall rules for DMR ports (55443)
- [ ] DNS resolution between sites
- [ ] TLS certificates properly configured
```

### Subscription Management

```markdown
## Topic Hierarchy for Multi-Site

Recommended structure:
  <region>/<domain>/<entity>/<action>

Examples:
  us-east/orders/order/created
  eu-west/orders/order/created
  
Consumers subscribe to:
  - Local region: us-east/orders/>
  - All regions: */orders/>
```

### Replication Strategies

| Strategy | Use Case | Configuration |
|----------|----------|---------------|
| Full Mesh | Small clusters | All nodes linked |
| Hub-Spoke | Central control | Hub links all spokes |
| Ring | Redundancy | Each node has 2 neighbors |
| Partial | Selective | Only needed links |

## Reference Links

- **DMR Overview**: https://docs.solace.com/DMR/DMR-Overview.htm
- **Event Mesh**: https://docs.solace.com/Event-Mesh/Event-Mesh-Overview.htm
- **DMR Configuration**: https://docs.solace.com/DMR/Configuring-DMR.htm

## Further Reading

- [Multi-Cloud Patterns](./references/multi-cloud.md)
- [DR Configuration](./references/disaster-recovery.md)
- [Performance Tuning](./references/dmr-performance.md)
