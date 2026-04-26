---
name: solace-event-portal
description: 'Solace Event Portal guidance for EDA design and governance. Use for Designer tool, creating applications/events/schemas, Catalog exploration, Runtime Event Manager, Event Portal REST API, AsyncAPI generation, KPI Dashboard, and AI Design Assistant.'
argument-hint: 'designer, catalog, runtime, asyncapi, or schema'
---

# Solace Event Portal

Expert guidance for using Solace Event Portal - a cloud-based event management tool for designing, discovering, and governing event-driven architectures.

## When to Use

- **Designing EDA**: Creating applications, events, and schemas in Designer
- **Event Discovery**: Exploring existing events in the Catalog
- **Runtime Management**: Connecting Event Portal to live brokers
- **API Integration**: Using Event Portal REST APIs
- **AsyncAPI**: Generating or importing AsyncAPI specifications
- **Governance**: Managing event schemas and versions

## Core Concepts

### Event Portal Components

| Component | Purpose |
|-----------|---------|
| **Designer** | Create and model EDA objects (applications, events, schemas) |
| **Catalog** | Search and discover existing EDA assets |
| **Runtime Event Manager** | Model topology using discovered runtime data |
| **Event Broker Connections** | Connect to brokers for push/discovery |
| **KPI Dashboard** | View event usage metrics |

### Object Hierarchy

```
Application Domain
├── Application
│   ├── Events Published
│   │   └── Event
│   │       └── Schema (version)
│   └── Events Subscribed
│       └── Event
└── Another Application
```

## Designer Tool

### Creating an Application Domain

1. Navigate to **Designer** → **Application Domains**
2. Click **+ Add Application Domain**
3. Enter:
   - **Name**: `retail-orders` (lowercase, hyphenated)
   - **Description**: Business context and ownership
   - **Topic Domain**: Root topic prefix (e.g., `acme/retail/orders`)

### Creating an Application

```yaml
# Application definition
name: order-service
description: Handles order creation and lifecycle
type: Application
version: 1.0.0

# Event relationships
publishes:
  - OrderCreated
  - OrderUpdated
  - OrderCancelled
  
subscribes:
  - PaymentReceived
  - InventoryReserved
```

### Creating Events

```yaml
# Event definition
name: OrderCreated
description: Emitted when a new order is placed
version: 1.2.0
topic: acme/retail/orders/{region}/created/v1/{orderId}

# Topic variables
topic_variables:
  - name: region
    description: Geographic region (us, eu, apac)
    enum: [us, eu, apac]
  - name: orderId
    description: Unique order identifier
    
# Schema reference
schema: OrderCreatedPayload
schema_version: 1.2.0
```

### Creating Schemas

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "OrderCreatedPayload",
  "type": "object",
  "properties": {
    "orderId": {
      "type": "string",
      "description": "Unique order identifier"
    },
    "customerId": {
      "type": "string"
    },
    "items": {
      "type": "array",
      "items": {
        "$ref": "#/$defs/OrderItem"
      }
    },
    "totalAmount": {
      "type": "number"
    },
    "currency": {
      "type": "string",
      "enum": ["USD", "EUR", "GBP"]
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["orderId", "customerId", "items", "totalAmount", "currency", "createdAt"],
  "$defs": {
    "OrderItem": {
      "type": "object",
      "properties": {
        "sku": { "type": "string" },
        "quantity": { "type": "integer" },
        "price": { "type": "number" }
      },
      "required": ["sku", "quantity", "price"]
    }
  }
}
```

## Event Portal REST API

### Authentication

```bash
# Generate API token in Cloud Console → Token Management
export EP_API_TOKEN="your-api-token"

# Base URL
EP_BASE_URL="https://api.solace.cloud/api/v2/architecture"
```

### Common Operations

#### List Application Domains

```bash
curl -X GET "${EP_BASE_URL}/applicationDomains" \
  -H "Authorization: Bearer ${EP_API_TOKEN}" \
  -H "Content-Type: application/json"
```

#### Create an Event

```bash
curl -X POST "${EP_BASE_URL}/events" \
  -H "Authorization: Bearer ${EP_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "OrderCreated",
    "applicationDomainId": "abc123",
    "shared": true,
    "description": "Emitted when order is created"
  }'
```

#### Create Event Version with Schema

```bash
curl -X POST "${EP_BASE_URL}/eventVersions" \
  -H "Authorization: Bearer ${EP_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "eventId": "evt-123",
    "version": "1.0.0",
    "schemaVersionId": "schema-456",
    "deliveryDescriptor": {
      "brokerType": "solace",
      "address": {
        "addressType": "topic",
        "addressLevels": [
          {"addressLevelType": "literal", "name": "acme"},
          {"addressLevelType": "literal", "name": "orders"},
          {"addressLevelType": "variable", "name": "region"},
          {"addressLevelType": "literal", "name": "created"}
        ]
      }
    }
  }'
```

### API Reference

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/applicationDomains` | GET/POST | List/create domains |
| `/applications` | GET/POST | List/create applications |
| `/events` | GET/POST | List/create events |
| `/eventVersions` | GET/POST | List/create event versions |
| `/schemas` | GET/POST | List/create schemas |
| `/schemaVersions` | GET/POST | List/create schema versions |
| `/eventApis` | GET/POST | Event API products |

**Full API Reference**: https://api.solace.dev/cloud/reference

## AsyncAPI Integration

### Export AsyncAPI from Event Portal

1. Navigate to application in Designer
2. Click **Export** → **AsyncAPI**
3. Select version and format (YAML/JSON)

### Generated AsyncAPI Spec

```yaml
asyncapi: 2.6.0
info:
  title: Order Service
  version: 1.0.0
  description: Handles order creation and lifecycle

channels:
  acme/retail/orders/{region}/created/v1/{orderId}:
    parameters:
      region:
        schema:
          type: string
          enum: [us, eu, apac]
      orderId:
        schema:
          type: string
    publish:
      operationId: publishOrderCreated
      message:
        $ref: '#/components/messages/OrderCreated'

components:
  messages:
    OrderCreated:
      name: OrderCreated
      title: Order Created Event
      contentType: application/json
      payload:
        $ref: '#/components/schemas/OrderCreatedPayload'
  
  schemas:
    OrderCreatedPayload:
      type: object
      properties:
        orderId:
          type: string
        # ... full schema
```

### Import AsyncAPI to Event Portal

```bash
curl -X POST "${EP_BASE_URL}/asyncApis/import" \
  -H "Authorization: Bearer ${EP_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationDomainId": "domain-123",
    "asyncApiSpecification": "... yaml content ..."
  }'
```

## Runtime Event Manager

### Connecting to Event Brokers

1. Navigate to **Runtime Event Manager** → **Event Broker Connections**
2. Click **+ Add Connection**
3. Configure:
   - **Name**: Connection name
   - **Type**: Solace Cloud / Software Broker
   - **SEMP Credentials**: Admin credentials for discovery

### Runtime Discovery

Once connected, Event Portal can:
- Discover existing queues, topics, subscriptions
- Import runtime topology into Designer
- Detect schema from message payloads
- Track drift between design and runtime

### Pushing Configurations

```bash
# Push application configuration to broker
curl -X POST "${EP_BASE_URL}/runtimeEventManager/push" \
  -H "Authorization: Bearer ${EP_API_TOKEN}" \
  -d '{
    "applicationVersionId": "app-v-123",
    "targetBrokerConnectionId": "conn-456",
    "dryRun": true
  }'
```

## AI Design Assistant

Event Portal includes an AI assistant for rapid EDA design:

1. Click **AI Design Assistant** in Designer
2. Describe your domain: "I need an e-commerce order system with orders, payments, and inventory"
3. AI generates:
   - Application domain structure
   - Applications with pub/sub relationships
   - Events with topic hierarchies
   - Sample schemas

## Best Practices

### Naming Conventions

| Object | Convention | Example |
|--------|------------|---------|
| Application Domain | `<business-area>-<capability>` | `retail-orders` |
| Application | `<domain>-<function>` | `order-service` |
| Event | `<Entity><Action>` (PascalCase) | `OrderCreated` |
| Schema | `<EventName>Payload` | `OrderCreatedPayload` |
| Topic | `<org>/<domain>/<entity>/<action>/v<N>` | `acme/retail/orders/created/v1` |

### Versioning Strategy

1. **Schema Versioning**: Use semantic versioning (1.0.0)
2. **Topic Versioning**: Include version in topic (`/v1/`, `/v2/`)
3. **Breaking Changes**: New major version, new topic
4. **Backward Compatible**: Minor version bump only

### Governance Workflow

```
Draft → Review → Released → Deprecated → Retired
  │        │         │           │
  └────────┴────Promote──────────┘
```

## Reference Links

- **Event Portal Documentation**: https://docs.solace.com/Cloud/Event-Portal/event-portal-lp.htm
- **Event Portal REST API**: https://api.solace.dev/cloud/reference
- **Codelab**: https://codelabs.solace.dev/codelabs/design-code-deploy-with-event-portal/ (90 min)
- **Quick Start**: https://solace.com/products/portal/event-portal-getting-started/

## Further Reading

- [Designer Reference](./references/designer.md)
- [Catalog Search](./references/catalog.md)
- [Runtime Integration](./references/runtime.md)
- [API Examples](./references/api-examples.md)
