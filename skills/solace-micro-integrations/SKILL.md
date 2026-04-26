---
name: solace-micro-integrations
description: 'Solace Micro-Integrations guidance for event-driven data integration. Use for configuring source/target connectors, Kafka/database/S3/Salesforce bridges, message transformation, managed vs self-managed deployment, Docker/Kubernetes deployment, and Integration Hub connectors.'
argument-hint: 'source, target, kafka, database, transformation, or deployment'
---

# Solace Micro-Integrations

Expert guidance for Solace Micro-Integrations - lightweight integration modules that connect enterprise systems to Solace event brokers.

## When to Use

- **Data Integration**: Connecting databases, Kafka, cloud services to Solace
- **Event Sourcing**: Ingesting data from external systems into the event mesh
- **Event Delivery**: Sending events to target systems
- **Message Transformation**: Converting between formats and protocols
- **Deployment**: Setting up managed or self-managed integrations

## Core Concepts

### What Are Micro-Integrations?

Micro-Integrations are small, purpose-built integration modules that:
- Connect external systems to Solace event brokers
- Transform messages between formats
- Enable real-time data exchange
- Replace monolithic integration patterns

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Source System  │────▶│ Micro-Integration│────▶│  Event Broker   │
│  (Kafka, DB)    │     │  (Transform)    │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Event Broker   │────▶│ Micro-Integration│────▶│  Target System  │
│                 │     │  (Transform)    │     │  (S3, Database) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### Deployment Options

| Type | Description | Management |
|------|-------------|------------|
| **Managed (Solace Cloud)** | Fully managed, infrastructure handled by Solace | Zero infrastructure |
| **Self-Managed** | Deploy on your infrastructure (Docker, K8s, VM) | Full control |

## Managed Micro-Integrations (Solace Cloud)

### Creating a Source Integration

1. Navigate to **Solace Cloud Console** → **Micro-Integrations**
2. Click **+ Create Integration**
3. Select **Source** (data flows TO your event broker)
4. Choose connector type:
   - Kafka
   - Amazon S3
   - Salesforce
   - Database (CDC)

### Kafka Source Configuration

```yaml
# Solace Cloud → Micro-Integrations → Create
name: kafka-orders-source
type: source
connector: kafka

source:
  bootstrapServers: kafka-cluster:9092
  topic: orders
  consumerGroup: solace-integration
  security:
    protocol: SASL_SSL
    mechanism: PLAIN
    username: ${KAFKA_USER}
    password: ${KAFKA_PASSWORD}

destination:
  eventBrokerService: my-broker-service
  topic: kafka/orders/ingested

transformation:
  format: passthrough  # or json, avro
```

### Creating a Target Integration

1. Select **Target** (data flows FROM your event broker)
2. Choose destination:
   - Kafka
   - Amazon S3
   - Snowflake
   - Database

### S3 Target Configuration

```yaml
name: events-to-s3
type: target
connector: s3

source:
  eventBrokerService: my-broker-service
  subscription: events/archive/>

destination:
  bucket: my-events-bucket
  region: us-east-1
  prefix: events/
  fileFormat: json
  partitioning:
    type: time
    pattern: year={yyyy}/month={MM}/day={dd}
  credentials:
    accessKeyId: ${AWS_ACCESS_KEY}
    secretAccessKey: ${AWS_SECRET_KEY}
```

## Self-Managed Micro-Integrations

### Installation

```bash
# Download connector package
wget https://solace.com/downloads/connectors/kafka-connector-2.x.x.tar.gz

# Extract
tar -xzf kafka-connector-2.x.x.tar.gz
cd kafka-connector

# Configure
cp config/application.properties.template config/application.properties
```

### Docker Deployment

```yaml
# docker-compose.yml
version: '3.8'
services:
  kafka-connector:
    image: solace/kafka-connector:latest
    environment:
      - SOLACE_HOST=tcp://broker:55555
      - SOLACE_VPN=default
      - SOLACE_USERNAME=admin
      - SOLACE_PASSWORD=admin
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    volumes:
      - ./config:/app/config
    restart: unless-stopped
```

### Kubernetes Deployment

```yaml
# kafka-connector-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-connector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kafka-connector
  template:
    metadata:
      labels:
        app: kafka-connector
    spec:
      containers:
      - name: connector
        image: solace/kafka-connector:latest
        env:
        - name: SOLACE_HOST
          valueFrom:
            secretKeyRef:
              name: solace-credentials
              key: host
        - name: SOLACE_USERNAME
          valueFrom:
            secretKeyRef:
              name: solace-credentials
              key: username
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

## Common Connectors

### Kafka Connector

**Source (Kafka → Solace)**:
```properties
# Source connector properties
sol.host=tcp://broker:55555
sol.vpn_name=default
sol.username=admin
sol.password=admin

kafka.bootstrap.servers=kafka:9092
kafka.consumer.group.id=solace-connector
kafka.topics=orders,payments

# Mapping
sol.topics=kafka/${kafka.topic}
sol.queue=kafka-ingress-queue
```

**Sink (Solace → Kafka)**:
```properties
# Sink connector properties
kafka.bootstrap.servers=kafka:9092
kafka.topic.default=solace-events

sol.host=tcp://broker:55555
sol.vpn_name=default
sol.queue=kafka-egress-queue
sol.subscription.topics=events/>
```

### Database Connector (CDC)

```yaml
# PostgreSQL CDC Source
name: postgres-cdc-source
connector: debezium-postgres

source:
  hostname: postgres-host
  port: 5432
  database: orders_db
  username: ${DB_USER}
  password: ${DB_PASSWORD}
  tables:
    - public.orders
    - public.customers

destination:
  topic: cdc/postgres/{schema}/{table}
  
transformation:
  format: json
  includeSchema: true
```

### REST Webhook Target

```yaml
name: webhook-target
connector: rest

source:
  subscription: alerts/critical/>

destination:
  url: https://api.pagerduty.com/events
  method: POST
  headers:
    Authorization: "Token ${PAGERDUTY_TOKEN}"
    Content-Type: application/json
  
transformation:
  template: |
    {
      "routing_key": "${ROUTING_KEY}",
      "event_action": "trigger",
      "payload": {
        "summary": "${message.text}",
        "severity": "critical",
        "source": "solace-events"
      }
    }
```

## Message Transformation

### Format Conversion

```yaml
transformation:
  # Input format
  inputFormat: avro
  inputSchemaRegistry: http://schema-registry:8081
  
  # Output format
  outputFormat: json
  
  # Field mapping
  mappings:
    - source: $.user_id
      target: $.userId
    - source: $.created_at
      target: $.timestamp
      transform: dateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'")
```

### Data Enrichment

```yaml
transformation:
  enrichment:
    - type: lookup
      source: $.customer_id
      lookup:
        type: rest
        url: https://api.example.com/customers/${value}
        cache:
          enabled: true
          ttl: 3600
      target: $.customer_details
```

### Filtering

```yaml
transformation:
  filter:
    # Only process orders over $100
    condition: "$.amount > 100"
    
  # Or use CEL expression
  filterExpression: 'message.amount > 100 && message.status == "PENDING"'
```

## Monitoring & Observability

### Metrics

```properties
# Enable metrics endpoint
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.server.port=8081

# Prometheus metrics
management.metrics.export.prometheus.enabled=true
```

### Health Checks

```bash
# Check connector health
curl http://localhost:8081/actuator/health

# Response
{
  "status": "UP",
  "components": {
    "solace": { "status": "UP" },
    "kafka": { "status": "UP" }
  }
}
```

## Best Practices

### Design Principles

1. **Single Responsibility**: One connector per integration flow
2. **Idempotency**: Design for at-least-once delivery
3. **Error Handling**: Configure dead-letter queues
4. **Monitoring**: Enable health checks and metrics

### Performance Tuning

```properties
# Batch processing
sol.publisher.batch.size=100
sol.publisher.batch.delay.ms=50

# Parallelism
kafka.consumer.max.poll.records=500
sol.consumer.flow.window.size=255
```

### High Availability

```yaml
# Run multiple instances
replicas: 3

# Configure consumer groups
kafka.consumer.group.id: solace-connector

# Enable broker failover
sol.host: tcp://primary:55555,tcp://backup:55555
sol.reconnect.retries: -1
```

## Reference Links

- **Micro-Integrations Overview**: https://docs.solace.com/Micro-Integrations/Micro-Integrations.htm
- **Integration Hub**: https://solace.com/integration-hub/
- **Kafka Connector Documentation**: https://docs.solace.com/connectors/kafka-connector/
- **Codelab - Kafka Connectors**: https://codelabs.solace.dev/codelabs/kafka-connectors/ (97 min)

## Further Reading

- [Kafka Connector Reference](./references/kafka-connector.md)
- [Database CDC Patterns](./references/database-cdc.md)
- [Transformation Examples](./references/transformations.md)
- [Deployment Best Practices](./references/deployment.md)
