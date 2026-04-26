---
name: solace-event-design-review
description: 'Solace event schema and design audit. Use for reviewing AsyncAPI specifications, event payload structure, schema evolution patterns, event versioning, and identifying event design anti-patterns.'
argument-hint: 'asyncapi, schema, versioning, or evolution'
---

# Solace Event Design Review

Expert guidance for auditing event schemas and design patterns.

## When to Use

- **Schema Review**: Validate event payload design
- **AsyncAPI Audit**: Review API specifications
- **Version Strategy**: Check schema evolution
- **Compatibility**: Verify backward/forward compatibility

## Event Design Checklist

### Event Structure

```markdown
## Event Structure Checklist

### Naming
- [ ] Event name is past tense (OrderCreated, not CreateOrder)
- [ ] Name clearly describes what happened
- [ ] Follows domain naming conventions
- [ ] Consistent with other events in domain

### Payload Design
- [ ] Contains all necessary context
- [ ] No unnecessary data (minimal payload)
- [ ] Uses standard data types
- [ ] Timestamps in ISO 8601 format
- [ ] IDs are strings (not numbers)

### Metadata
- [ ] Includes event ID (unique)
- [ ] Includes correlation ID
- [ ] Has timestamp
- [ ] Version information present
- [ ] Source/producer identified
```

### Schema Best Practices

```markdown
## Schema Quality Checklist

### Type Safety
- [ ] All fields have explicit types
- [ ] Enums defined for constrained values
- [ ] Nullable fields marked explicitly
- [ ] Required fields specified

### Documentation
- [ ] All fields documented
- [ ] Examples provided
- [ ] Validation rules documented
- [ ] Business meaning explained

### Evolution
- [ ] New fields are optional
- [ ] Deprecation marked clearly
- [ ] Breaking changes versioned
- [ ] Default values specified
```

## Event Schema Examples

### Good Event Design

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "title": "OrderCreatedEvent",
  "description": "Published when a new order is placed successfully",
  "required": ["eventId", "eventTime", "orderId", "customerId", "items", "total"],
  "properties": {
    "eventId": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier for this event instance"
    },
    "eventTime": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when event occurred"
    },
    "correlationId": {
      "type": "string",
      "description": "Trace ID for distributed tracing"
    },
    "eventVersion": {
      "type": "string",
      "const": "1.0",
      "description": "Schema version of this event"
    },
    "orderId": {
      "type": "string",
      "description": "Unique order identifier"
    },
    "customerId": {
      "type": "string",
      "description": "Customer who placed the order"
    },
    "items": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/OrderItem"
      },
      "minItems": 1,
      "description": "Items in the order"
    },
    "total": {
      "$ref": "#/definitions/Money",
      "description": "Order total amount"
    },
    "status": {
      "type": "string",
      "enum": ["PENDING", "CONFIRMED", "PROCESSING"],
      "description": "Order status at time of creation"
    }
  },
  "definitions": {
    "OrderItem": {
      "type": "object",
      "required": ["productId", "quantity", "price"],
      "properties": {
        "productId": {"type": "string"},
        "quantity": {"type": "integer", "minimum": 1},
        "price": {"$ref": "#/definitions/Money"}
      }
    },
    "Money": {
      "type": "object",
      "required": ["amount", "currency"],
      "properties": {
        "amount": {"type": "number"},
        "currency": {"type": "string", "pattern": "^[A-Z]{3}$"}
      }
    }
  }
}
```

### Bad Event Design (Anti-Patterns)

```json
{
  // ❌ No schema version
  // ❌ No event metadata
  // ❌ Numeric IDs (should be strings)
  // ❌ Non-standard date format
  // ❌ Abbreviated field names
  // ❌ No type constraints
  
  "oid": 12345,
  "cid": 67890,
  "dt": "03/15/2024",
  "itms": [
    {"pid": 1, "q": 2, "p": 29.99}
  ],
  "tot": 59.98,
  "st": 1
}
```

## AsyncAPI Review

### AsyncAPI Audit Checklist

```markdown
## AsyncAPI Specification Checklist

### Info Section
- [ ] Title and description present
- [ ] Version specified
- [ ] Contact information included
- [ ] License defined

### Channels
- [ ] All topics documented
- [ ] Operations clearly defined
- [ ] Bindings appropriate for Solace
- [ ] Parameters defined for dynamic topics

### Messages
- [ ] All messages have schemas
- [ ] Content type specified
- [ ] Examples provided
- [ ] Headers documented

### Schemas
- [ ] All referenced schemas exist
- [ ] Schemas are complete
- [ ] No circular references
- [ ] Common types factored out
```

### Sample AsyncAPI Specification

```yaml
asyncapi: '2.6.0'
info:
  title: Order Events API
  version: '1.0.0'
  description: Events published by the Order Service
  contact:
    name: Order Team
    email: orders@example.com
  license:
    name: Internal

servers:
  production:
    url: mr-connection-xxx.messaging.solace.cloud:55443
    protocol: solace
    description: Production Solace Cloud

channels:
  acme/orders/order/created/v1:
    publish:
      operationId: onOrderCreated
      summary: Order was placed
      message:
        $ref: '#/components/messages/OrderCreatedEvent'
    bindings:
      solace:
        destinations:
          - destinationType: queue
            queue:
              name: order-events-queue
              accessType: non-exclusive

  acme/orders/order/updated/v1/{orderId}:
    parameters:
      orderId:
        schema:
          type: string
    publish:
      operationId: onOrderUpdated
      message:
        $ref: '#/components/messages/OrderUpdatedEvent'

components:
  messages:
    OrderCreatedEvent:
      contentType: application/json
      headers:
        type: object
        properties:
          correlationId:
            type: string
      payload:
        $ref: '#/components/schemas/OrderCreatedPayload'
      examples:
        - payload:
            eventId: "evt-123"
            orderId: "ord-456"
            total: {amount: 99.99, currency: "USD"}

  schemas:
    OrderCreatedPayload:
      type: object
      required: [eventId, eventTime, orderId, total]
      properties:
        eventId:
          type: string
          format: uuid
        eventTime:
          type: string
          format: date-time
        orderId:
          type: string
        total:
          $ref: '#/components/schemas/Money'

    Money:
      type: object
      properties:
        amount:
          type: number
        currency:
          type: string
```

## Schema Evolution

### Compatibility Rules

| Change Type | Backward | Forward |
|-------------|----------|---------|
| Add optional field | ✅ | ✅ |
| Add required field | ❌ | ✅ |
| Remove optional field | ✅ | ❌ |
| Remove required field | ✅ | ❌ |
| Rename field | ❌ | ❌ |
| Change field type | ❌ | ❌ |

### Version Strategy

```yaml
# Option 1: Version in topic (recommended)
channels:
  acme/orders/order/created/v1:
    # V1 schema
  acme/orders/order/created/v2:
    # V2 schema with breaking changes

# Option 2: Version in payload
payload:
  properties:
    eventVersion:
      type: string
      enum: ["1.0", "2.0"]
```

### Migration Pattern

```python
# Consumer handles multiple versions
def handle_order_created(message):
    version = message.headers.get('eventVersion', '1.0')
    
    if version == '1.0':
        payload = OrderCreatedV1.parse(message.payload)
        # Transform to V2 if needed
        payload = migrate_v1_to_v2(payload)
    else:
        payload = OrderCreatedV2.parse(message.payload)
    
    process_order(payload)
```

## Event Design Anti-Patterns

### 1. Fat Events

```json
// ❌ BAD: Contains entire order with all relationships
{
  "orderId": "123",
  "customer": {
    "id": "456",
    "name": "John Doe",
    "email": "john@email.com",
    "addresses": [...],      // Too much data!
    "orderHistory": [...]    // Way too much!
  },
  "products": [
    {
      "id": "789",
      "name": "Widget",
      "fullDescription": "...",  // Unnecessary
      "allInventoryLevels": [...] // Irrelevant
    }
  ]
}

// ✅ GOOD: Contains just IDs and essential data
{
  "orderId": "123",
  "customerId": "456",
  "items": [{"productId": "789", "quantity": 2}],
  "total": {"amount": 99.99, "currency": "USD"}
}
```

### 2. Missing Event Context

```json
// ❌ BAD: No metadata
{
  "orderId": "123",
  "status": "SHIPPED"
}

// ✅ GOOD: Full context
{
  "eventId": "evt-abc123",
  "eventTime": "2024-01-15T10:30:00Z",
  "correlationId": "trace-xyz",
  "eventType": "OrderShipped",
  "eventVersion": "1.0",
  "orderId": "123",
  "status": "SHIPPED",
  "shipmentId": "ship-456"
}
```

### 3. Command Disguised as Event

```json
// ❌ BAD: This is a command, not an event
{
  "action": "UPDATE_ORDER",
  "orderId": "123",
  "newStatus": "CANCELLED"
}

// ✅ GOOD: Event describes what happened
{
  "eventType": "OrderCancelled",
  "orderId": "123",
  "cancelledAt": "2024-01-15T10:30:00Z",
  "reason": "Customer request",
  "refundInitiated": true
}
```

## Event Design Review Report

```markdown
# Event Design Review Report

**Domain**: [name]
**Date**: [date]
**Reviewer**: [name]

## Summary
- Events Reviewed: [count]
- Compliant: [count] ([%])
- Issues Found: [count]

## Event Inventory
| Event | Version | Score | Issues |
|-------|---------|-------|--------|
| OrderCreated | 1.0 | 90 | - |
| OrderUpdated | 1.0 | 75 | Missing timestamp |

## Schema Quality
| Criterion | Pass | Fail |
|-----------|------|------|
| Has event ID | X | X |
| Has timestamp | X | X |
| Types defined | X | X |
| Documented | X | X |

## Compatibility Analysis
| Event | Backward | Forward | Breaking Changes |
|-------|----------|---------|------------------|
| OrderCreated | ✅ | ✅ | None |
| OrderUpdated | ❌ | ✅ | Field removed |

## Recommendations
1. Add missing metadata fields
2. Version event with breaking change
3. Add documentation to schemas
```

## Reference Links

- **AsyncAPI Solace**: https://docs.solace.com/AsyncAPI/AsyncAPI-Introduction.htm
- **Event Portal**: https://docs.solace.com/Cloud/Event-Portal/event-portal.htm
- **Schema Registry**: https://docs.solace.com/Event-Portal/Schema-Registry.htm

## Further Reading

- [AsyncAPI Best Practices](./references/asyncapi.md)
- [Schema Evolution](./references/schema-evolution.md)
- [Event Naming](./references/event-naming.md)
