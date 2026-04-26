---
name: solace-topic-architecture
description: 'Solace topic design and governance guidance. Use for hierarchical topic design patterns, wildcard subscriptions (* and >), topic taxonomy best practices, event naming conventions, multi-tenant isolation, and topic versioning strategies.'
argument-hint: 'hierarchy, wildcards, naming, multi-tenant, or versioning'
---

# Solace Topic Architecture

Expert guidance for designing effective topic hierarchies and governing topic namespaces in Solace PubSub+.

## When to Use

- **Designing Topic Hierarchy**: Creating topic structure for new systems
- **Naming Conventions**: Establishing consistent naming standards
- **Wildcard Strategy**: Optimal use of `*` and `>` wildcards
- **Multi-Tenancy**: Isolating topics across tenants
- **Governance**: Managing topic namespaces at scale

## Topic Fundamentals

### Official Topic Structure (Solace Best Practice)

A well-defined event topic has two main parts:

```
<Domain>/<Noun>/<Verb>/<Version>/<Properties...>
```

| Element | Description | Examples |
|---------|-------------|----------|
| **Domain** | Organizational element responsible for the system. Multiple levels ensure events don't collide and identify data owner. | `ops/flights`, `hr/payroll`, `finance/billing` |
| **Noun** | The type of object being acted on | `order`, `customer`, `inventory`, `payment` |
| **Verb** | Action taken or state changed (past tense) | `created`, `deleted`, `exceeded`, `pickedUp`, `paid` |
| **Version** | Event version for routing and schema changes | `v1`, `v2` |
| **Properties** | Dynamic fields for routing/filtering (ordered by cardinality: least to most specific) | `{region}`, `{orderId}`, `{customerId}` |

### Example Topic Taxonomy

```
# Airline flight operations example
ops/flights/flight/{status}/v1/{flightNumber}/{origin}/{destination}

# Concrete examples:
ops/flights/flight/boarding/v1/ea1234/jfk/ord
ops/flights/flight/delayed/v1/ea9999/yow/sin
ops/flights/flight/wheelsUp/v1/ea1010/ewr/ord

# Subscriber patterns:
ops/flights/flight/wheelsUp/*/jfk/>    # All flights leaving JFK
ops/flights/flight/delayed/>           # All delayed flights
ops/flights/flight/*/ea1010/>          # All events for flight EA 1010
```

### Topic Structure

```
<organization>/<domain>/<subdomain>/<entity>/<action>/<version>/<key>
```

Example:
```
acme/retail/orders/order/created/v1/ord-12345
  │     │      │      │      │     │     │
  │     │      │      │      │     │     └── Key/ID (optional)
  │     │      │      │      │     └── Version
  │     │      │      │      └── Action (verb)
  │     │      │      └── Entity (noun)
  │     │      └── Subdomain
  │     └── Domain
  └── Organization
```

### Wildcard Types

| Wildcard | Matches | Example |
|----------|---------|---------|
| `*` | Single level exactly | `acme/*/orders` matches `acme/retail/orders` |
| `>` | One or more levels | `acme/retail/>` matches `acme/retail/orders/created` |

## Design Patterns

### Pattern 1: Domain-Driven Topics

```
<org>/<bounded-context>/<aggregate>/<event-type>/v<N>

Examples:
acme/inventory/product/created/v1
acme/inventory/product/updated/v1
acme/inventory/stock/adjusted/v1
acme/orders/order/placed/v1
acme/orders/order/shipped/v1
acme/payments/payment/received/v1
```

### Pattern 2: Entity-Action-Outcome

```
<org>/<domain>/<entity>/<action>/<outcome>

Examples:
acme/retail/order/create/success
acme/retail/order/create/failed
acme/retail/order/update/success
acme/retail/payment/process/success
acme/retail/payment/process/declined
```

### Pattern 3: Region-Aware Topics

```
<org>/<region>/<domain>/<entity>/<action>/v<N>

Examples:
acme/us-east/orders/order/created/v1
acme/eu-west/orders/order/created/v1
acme/apac/orders/order/created/v1

# Subscribe to all regions
acme/*/orders/order/created/v1
```

### Pattern 4: Multi-Tenant Topics

```
<org>/<tenant-id>/<domain>/<entity>/<action>/v<N>

Examples:
acme/tenant-abc/orders/order/created/v1
acme/tenant-xyz/orders/order/created/v1

# ACL can restrict access by tenant prefix
```

## Naming Conventions

### Recommended Standards

| Element | Convention | Examples |
|---------|------------|----------|
| Levels | kebab-case | `order-service`, `user-profile` |
| Entities | Singular nouns | `order`, `user`, `product` |
| Actions | Past tense verbs | `created`, `updated`, `deleted` |
| Versions | `v` + number | `v1`, `v2` |
| Keys | Original format | `ord-12345`, `uuid` |

### Naming Rules

✅ **DO**:
```
acme/retail/order/created/v1           # Consistent kebab-case
acme/retail/inventory/stock-adjusted/v1 # Compound words hyphenated
acme/retail/user/profile-updated/v1    # Clear action
```

❌ **DON'T**:
```
ACME/Retail/ORDER/Created/V1           # Inconsistent casing
acme.retail.order.created              # Using dots (reserved)
acme/retail/orders/OrderCreated        # CamelCase, plural
acme/retail/order/update               # Ambiguous (command or event?)
```

## Wildcard Strategies

### Efficient Subscriptions

```
# Good: Bounded wildcards
acme/retail/*/order/created/v1         # All retail subdomains
acme/retail/orders/>                   # All order events

# Bad: Overly broad (performance impact)
>                                      # Everything!
acme/>                                 # All acme events

# Better alternative to broad wildcards
acme/retail/orders/*                   # Just immediate children
acme/retail/orders/order/*             # Order entity events only
```

### Subscription Patterns by Use Case

| Use Case | Subscription Pattern |
|----------|---------------------|
| Specific event | `acme/retail/order/created/v1` |
| All order events | `acme/retail/order/>/v1` |
| All v1 events in domain | `acme/retail/>/v1` |
| Specific tenant | `acme/tenant-abc/>` |
| All regions, one event | `acme/*/orders/order/created/v1` |

## Queue Topic Subscriptions

### Mapping Topics to Queues

```yaml
# Queue: order-processor
subscriptions:
  - acme/retail/order/created/v1
  - acme/retail/order/updated/v1
  - acme/retail/order/cancelled/v1

# Queue: analytics-ingestion (broader)
subscriptions:
  - acme/retail/order/>/v1     # All order events
  - acme/retail/inventory/>/v1 # All inventory events

# Queue: audit-log (very broad - use carefully)
subscriptions:
  - acme/retail/>              # All retail events
```

### Partitioned Queue Topics

```
# For ordered processing with parallelism
Topic: acme/retail/order/created/v1/{orderId}

# Partition key extracted from topic
# Messages with same orderId go to same partition
```

## Version Management

### Versioning Strategy

```
# Initial version
acme/retail/order/created/v1

# Backward-compatible change (additive)
# Continue using v1, update schema with optional fields

# Breaking change
acme/retail/order/created/v2
# Keep v1 running during migration period

# Deprecation flow
v1: Active
v2: Active (new consumers use this)
v1: Deprecated (no new consumers)
v1: Retired (no publishers)
```

### Version in Schema vs Topic

```
# Version in topic (recommended for routing)
acme/retail/order/created/v1
acme/retail/order/created/v2

# Benefits:
# - Consumers can subscribe to specific versions
# - Easy to route old/new versions differently
# - Clear visibility of versions in use
```

## Multi-Tenant Isolation

### Tenant Prefix Pattern

```
# Structure
<org>/tenant/<tenant-id>/<domain>/<entity>/<action>/v<N>

# Examples
acme/tenant/abc-corp/orders/order/created/v1
acme/tenant/xyz-inc/orders/order/created/v1
```

### ACL Configuration

```json
{
  "aclProfileName": "tenant-abc-acl",
  "publishTopicExceptions": [
    {
      "publishTopicException": "acme/tenant/abc-corp/>",
      "publishTopicExceptionSyntax": "smf"
    }
  ],
  "subscribeTopicExceptions": [
    {
      "subscribeTopicException": "acme/tenant/abc-corp/>",
      "subscribeTopicExceptionSyntax": "smf"
    }
  ]
}
```

## Topic Governance

### Documentation Template

```yaml
# Topic Taxonomy Documentation
domain: retail
owner: retail-engineering@example.com

topics:
  - pattern: acme/retail/order/created/v1
    description: Published when a new order is placed
    publisher: order-service
    consumers:
      - inventory-service
      - notification-service
      - analytics
    schema: OrderCreatedEvent
    sla:
      latency: 100ms
      availability: 99.9%
    
  - pattern: acme/retail/order/updated/v1
    description: Published when order details change
    publisher: order-service
    consumers:
      - tracking-service
```

### Review Checklist

```markdown
## Topic Review Checklist

### Structure
- [ ] Follows organizational hierarchy pattern
- [ ] Maximum 7-8 levels deep
- [ ] Entity names are singular nouns
- [ ] Actions are past-tense verbs
- [ ] Version segment present

### Naming
- [ ] Consistent casing (kebab-case)
- [ ] No special characters except hyphens
- [ ] No PII in topic names
- [ ] Descriptive but concise

### Routing
- [ ] Wildcards are bounded appropriately
- [ ] No subscriber using `>` alone
- [ ] Queue subscriptions documented
- [ ] Partition keys identified if needed

### Governance
- [ ] Owner documented
- [ ] Schema referenced
- [ ] Consumers listed
- [ ] SLA defined
```

## Anti-Patterns (Official Solace Best Practices)

### ❌ Don't Use Message Properties for Filtering
```
# BAD: Using JMS selectors for filtering
# Selectors cause architectural problems and can't be used for
# access control, routing, or governance decisions

// Anti-pattern in JMS
String selector = "region = 'US' AND priority > 5";
consumer = session.createConsumer(queue, selector);

# GOOD: Encode filtering info in topic hierarchy
acme/retail/us/orders/high-priority/order/created/v1
acme/retail/eu/orders/normal/order/created/v1

# Then subscribe with wildcards
acme/retail/us/orders/high-priority/>  # US high-priority only
```

### ❌ Don't Include Environment Names in Topics
```
# BAD: Environment in topic (causes deployment problems)
dev/acme/retail/order/created/v1
qa/acme/retail/order/created/v1
prod/acme/retail/order/created/v1

# Problems:
# - Forces application code to change between environments
# - Requires changing ACLs, subscriptions, replay config per env
# - Prevents using Event Portal for promotion tracking

# GOOD: Same topics across all environments
acme/retail/order/created/v1

# Use separate VPNs/brokers per environment instead
```

### ❌ Don't Include Tracing Information in Topics
```
# BAD: TraceID in topic (wastes space, not useful for routing)
acme/retail/order/created/v1/trace-abc123-def456

# GOOD: Use Solace Distributed Tracing feature instead
# TraceID is automatically propagated in message headers
```

### ❌ Avoid Special Characters and Spaces
```
# BAD: Special characters
acme/retail/order created/v1       # Space
acme/retail/order_created/v1       # Underscore (wastes characters)
acme/retail/order*/v1              # Asterisk (subscription wildcard)
acme/retail/order>/v1              # Greater-than (subscription wildcard)
acme/retail/order!/v1              # Exclamation (exception prefix)

# GOOD: Use camelCase or kebab-case (camelCase preferred for efficiency)
acme/retail/orderCreated/v1        # camelCase (more efficient)
acme/retail/order-created/v1       # kebab-case (acceptable)
```

### ❌ Flat Topics
```
# Bad: No hierarchy
order-created
order-updated
inventory-updated

# Good: Proper hierarchy
acme/retail/order/created/v1
acme/retail/order/updated/v1
acme/inventory/stock/updated/v1
```

### ❌ PII in Topics
```
# Bad: Contains user data
acme/users/john.doe@email.com/notifications

# Good: Use ID reference
acme/users/notifications/user-12345
```

### ❌ Overly Broad Wildcards
```
# Bad: Subscribes to everything
subscription: >

# Good: Bounded subscription
subscription: acme/retail/orders/>
```

### ❌ Inconsistent Naming
```
# Bad: Mixed conventions
acme/Retail/orderCreated        # CamelCase
acme/retail/order_updated       # snake_case
acme/RETAIL/ORDER/DELETED       # UPPERCASE

# Good: Consistent
acme/retail/order/created/v1
acme/retail/order/updated/v1
acme/retail/order/deleted/v1
```

## Reference Links

- **Topic Best Practices**: https://docs.solace.com/Messaging/SMF-Topics/Topic-Architecture-Best-Practices.htm
- **Wildcard Subscriptions**: https://docs.solace.com/Messaging/SMF-Topics/Wildcard-Charaters-Topic-Subs.htm
- **Understanding Topics**: https://docs.solace.com/Messaging/SMF-Topics/What-Are-Topics.htm

## Further Reading

- [Naming Convention Examples](./references/naming-examples.md)
- [Multi-Tenant Patterns](./references/multi-tenant.md)
- [Migration Strategies](./references/version-migration.md)
