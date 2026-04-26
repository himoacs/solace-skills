---
name: solace-topic-review
description: 'Solace topic naming and hierarchy audit. Use for reviewing topic taxonomy, naming conventions, wildcard usage, multi-tenant isolation patterns, version management, and identifying topic design anti-patterns.'
argument-hint: 'audit or design'
---

# Solace Topic Review

Expert guidance for auditing topic naming conventions and hierarchy design.

## When to Use

- **Topic Audit**: Review existing topic taxonomy
- **Design Review**: Validate new topic hierarchy design
- **Governance**: Enforce naming standards
- **Optimization**: Identify inefficient topic patterns

## Review Checklist

### Structure and Hierarchy

```markdown
## Topic Structure Checklist

### Hierarchy Design
- [ ] Clear hierarchical organization
- [ ] Consistent depth (5-8 levels typical)
- [ ] Business domain at top level
- [ ] Entity/action pattern followed
- [ ] Version segment present

### Naming Conventions
- [ ] Consistent casing (kebab-case recommended)
- [ ] No special characters
- [ ] Meaningful names (not abbreviations)
- [ ] No PII in topic names
- [ ] Singular nouns for entities
- [ ] Past-tense verbs for events

### Documentation
- [ ] Topic taxonomy documented
- [ ] Owner assigned per domain
- [ ] Schema referenced for each topic
- [ ] Consumer/producer mapping exists
```

## Topic Naming Standards

### Recommended Pattern

```
<org>/<domain>/<subdomain>/<entity>/<action>/v<N>/<key>
```

### Scoring Matrix

| Criterion | Good (+2) | Acceptable (+1) | Poor (-1) | Bad (-2) |
|-----------|-----------|-----------------|-----------|----------|
| Casing | kebab-case | snake_case | Mixed | UPPERCASE |
| Depth | 5-7 levels | 4 or 8 | 3 or 9 | <3 or >9 |
| Entity names | Singular nouns | Plural nouns | Verbs | Unclear |
| Action names | Past tense | Present tense | Noun | Missing |
| Versioning | v1, v2 format | Version in name | No version | N/A |

### Examples

**Good Topics** (Score: +10):
```
acme/orders/retail/order/created/v1
acme/inventory/warehouse/stock/adjusted/v1
acme/payments/processing/payment/completed/v1
acme/shipping/logistics/shipment/dispatched/v1
```

**Poor Topics** (Score: -5):
```
ORDERS_CREATED                    # No hierarchy, uppercase
createOrder                       # Command, not event
acme.orders.created              # Wrong separator
acme/orders/OrderCreated/v1      # CamelCase
user/john.doe@email.com/events   # Contains PII
```

## Wildcard Analysis

### Subscription Audit

```markdown
## Wildcard Subscription Checklist

- [ ] No subscription to `>` alone
- [ ] `>` wildcard bounded by at least 2 levels
- [ ] `*` used appropriately for single-level
- [ ] Wildcard subscriptions documented
- [ ] Performance impact assessed
```

### Wildcard Patterns

| Pattern | Risk | Recommendation |
|---------|------|----------------|
| `>` | CRITICAL | Never use alone |
| `acme/>` | HIGH | Too broad for production |
| `acme/orders/>` | MEDIUM | Acceptable with monitoring |
| `acme/orders/retail/*` | LOW | Good - bounded |
| `acme/orders/retail/order/created/v1` | NONE | Best - specific |

### Finding Problematic Subscriptions

```bash
# List all subscriptions
curl -u admin:admin "$SEMP_URL/monitor/msgVpns/default/subscriptions" \
  | jq '.data[] | select(.topicName | test("^>$|^[^/]+/>$"))' \
  | jq '{topic: .topicName, queue: .queueName, type: "OVERLY_BROAD"}'
```

## Multi-Tenant Topic Patterns

### Recommended Pattern

```
<org>/tenant/<tenant-id>/<domain>/<entity>/<action>/v<N>

Example:
acme/tenant/cust-12345/orders/order/created/v1
```

### Tenant Isolation Audit

```markdown
## Multi-Tenant Checklist

- [ ] Tenant ID at fixed level in hierarchy
- [ ] ACL profiles per tenant
- [ ] Subscriptions scoped to tenant
- [ ] No cross-tenant topic access
- [ ] Wildcard cannot span tenants
```

### ACL Validation

```bash
# Verify tenant isolation in ACLs
curl -u admin:admin "$SEMP_URL/config/msgVpns/default/aclProfiles" | \
jq '.data[] | select(.publishTopicDefaultAction == "disallow") | {
  profile: .aclProfileName
}'

# Check publish exceptions are tenant-scoped
curl -u admin:admin "$SEMP_URL/config/msgVpns/default/aclProfiles/tenant-abc-acl/publishTopicExceptions" | \
jq '.data[] | {topic: .publishTopicException}'
```

## Version Management

### Versioning Audit

```markdown
## Version Management Checklist

- [ ] All topics include version segment
- [ ] Version format consistent (v1, v2)
- [ ] Old versions tracked
- [ ] Deprecation timeline documented
- [ ] Migration path defined
```

### Version Usage Analysis

```bash
# Find topics by version
curl -u admin:admin "$SEMP_URL/monitor/msgVpns/default/queues/*/subscriptions" | \
jq '.data[] | .subscriptionTopic' | \
grep -E '/v[0-9]+' | sort | uniq -c | sort -rn

# Example output:
#  45 acme/orders/*/v1
#  12 acme/orders/*/v2
#   3 acme/orders/*/v3
```

## Common Anti-Patterns

### 1. No Hierarchy

```
❌ orderCreated
❌ ORDER_UPDATED
❌ payment.completed

✅ acme/orders/order/created/v1
```

### 2. PII in Topics

```
❌ users/john.doe@email.com/notifications
❌ orders/customer/John Smith/updates

✅ users/notifications/user-12345
✅ orders/updates/customer-67890
```

### 3. Inconsistent Naming

```
❌ acme/Orders/OrderCreated      # CamelCase
❌ acme/orders/order_created     # snake_case
❌ ACME/ORDERS/CREATED           # UPPERCASE

✅ acme/orders/order/created/v1  # Consistent kebab-case
```

### 4. Missing Versions

```
❌ acme/orders/order/created     # No version!
❌ acme/orders/order/createdV2   # Version in action name

✅ acme/orders/order/created/v1
✅ acme/orders/order/created/v2
```

### 5. Commands as Topics

```
❌ acme/orders/createOrder       # Command, not event
❌ acme/inventory/updateStock    # Command, not event

✅ acme/orders/order/created/v1  # Event (past tense)
✅ acme/inventory/stock/updated/v1
```

## Topic Review Report Template

```markdown
# Topic Architecture Review Report

**System**: [name]
**Date**: [date]
**Reviewer**: [name]

## Summary
- Topics Reviewed: [count]
- Compliant: [count] ([%])
- Non-Compliant: [count] ([%])
- Critical Issues: [count]

## Scoring
| Category | Score | Max |
|----------|-------|-----|
| Hierarchy | X | 20 |
| Naming | X | 20 |
| Wildcards | X | 20 |
| Versioning | X | 20 |
| Documentation | X | 20 |
| **Total** | **X** | **100** |

## Findings

### Critical Issues
1. [Issue]: [topic pattern]
   - Impact: [description]
   - Fix: [recommendation]

### Non-Compliant Topics
| Topic | Issue | Recommended |
|-------|-------|-------------|
| [topic] | [issue] | [fix] |

### Wildcard Concerns
| Subscription | Risk Level | Recommendation |
|--------------|------------|----------------|
| [pattern] | [HIGH/MED/LOW] | [action] |

## Recommendations
1. [Priority 1]
2. [Priority 2]
3. [Priority 3]
```

## Automated Validation

### Topic Naming Validator

```python
import re

class TopicValidator:
    PATTERNS = {
        'hierarchy': r'^[a-z][a-z0-9-]*/[a-z][a-z0-9-]*/.*',
        'kebab-case': r'^[a-z0-9-/]+$',
        'version': r'/v\d+(/|$)',
        'no-pii': r'^(?!.*@)(?!.*\d{3}-\d{2}-\d{4}).*$'
    }

    def validate(self, topic: str) -> dict:
        results = {}
        for name, pattern in self.PATTERNS.items():
            results[name] = bool(re.match(pattern, topic))
        
        results['depth'] = 5 <= topic.count('/') <= 8
        results['score'] = sum(2 if v else -1 for v in results.values())
        return results

# Usage
validator = TopicValidator()
result = validator.validate("acme/orders/order/created/v1")
# {'hierarchy': True, 'kebab-case': True, 'version': True, 
#  'no-pii': True, 'depth': True, 'score': 10}
```

## Official Topic Architecture Checklist (from Solace Documentation)

### Topic Root Structure

```markdown
## Standard Topic Root Format

<Domain>/<Noun>/<Verb>/<Version>

| Element | Rules |
|---------|-------|
| Domain | Multiple levels OK (org/dept/app). Identifies data owner. |
| Noun | Type of object being acted on (order, customer, inventory) |
| Verb | Past tense action or state change (created, updated, exceeded) |
| Version | Format: v1, v2 (for routing and schema changes) |
```

### Topic Properties Ordering

```markdown
## Properties After Root (ordered by cardinality)

Properties should be ordered from LEAST specific to MOST specific.

Example - Sales order with city and order ID:
✅ GOOD: sales/order/created/v1/{city}/{orderId}
   (More cities than orders expected? Actually more orders, so reverse)
✅ BETTER: sales/order/created/v1/{city}/{orderId}

Example - Bus location updates:
mobile/bus/locUpdate/v1/{routeNumber}/{busNumber}/{location}
(Fewer routes → fewer buses per route → many locations per bus)
```

### Official Anti-Pattern Checklist

```markdown
## Things to AVOID (from Solace Best Practices)

### ❌ Message Properties for Filtering
- [ ] NOT using JMS selectors for routing/filtering
- [ ] Routing info is in topic hierarchy, not message properties
- [ ] Reason: Can't make ACL, governance, or routing decisions on properties

### ❌ Tracing Information in Topics
- [ ] TraceID/SpanID NOT in topic hierarchy
- [ ] Using Solace Distributed Tracing feature instead
- [ ] Reason: Wastes topic space, not useful for routing

### ❌ Environment Names in Topics
- [ ] dev/qa/prod NOT in topic names
- [ ] Same topics across all environments
- [ ] Environment separation via VPNs/brokers
- [ ] Reason: Forces code changes per environment, breaks CI/CD

### ❌ Special Characters
- [ ] No spaces in topics
- [ ] No * > ! characters (subscription wildcards)
- [ ] Prefer camelCase over snake_case (more efficient)
- [ ] Reason: Spaces waste characters, wildcards cause subscription issues

### ❌ Selectors Instead of Topics
- [ ] NOT using JMS selectors: session.createConsumer(queue, "region='US'")
- [ ] Using topic hierarchy: acme/us/orders/created/v1
- [ ] Reason: Selectors can't be used for access control
```

### Null Value Handling

```markdown
## Handling NULL Topic Levels

When a topic level may be null:
1. First consider: Is this level still useful for routing/ACL/governance?
2. If not, remove the level entirely
3. If yes, define a sentinel value

Recommended null values:
- Numeric: 0
- Enumeration: define default value
- Customer ID: 0000 (when no customer logged in)
- String: _null_ (if underscore cannot occur naturally)

IMPORTANT: Document the null value for each field!
AVOID: # * > ! characters in null values (special in subscriptions)
```

### Subscription Exception Pattern

```markdown
## Using Negative Subscriptions

For special handling of specific messages:

// Subscribe to all flights except EA1234
subscribe("flight/*")
subscribe("!flight/ex1234")  // Negative subscription

IMPORTANT:
- Only works for Guaranteed messaging
- Enable reject-msg-to-sender-on-no-subscription-match on client profile
- Better approach: encode handling instruction in topic level
```

### Event Documentation Checklist

```markdown
## Required Documentation Per Event

For each event topic, document:
- [ ] Topic pattern with variable placeholders
- [ ] Owner/responsible team
- [ ] Schema definition (JSON Schema, Avro, etc.)
- [ ] Value space for each topic level (enum, format, range)
- [ ] Null value if applicable
- [ ] Naming convention (camelCase recommended)
- [ ] Version history
- [ ] Consumer list
- [ ] SLA (latency, availability)

Tool: Use Solace Event Portal for:
- Event catalog and discovery
- Schema management
- Version tracking
- Deployment lifecycle
```

## Reference Links

- **Topic Best Practices**: https://docs.solace.com/Messaging/SMF-Topics/Topic-Architecture-Best-Practices.htm
- **Wildcards**: https://docs.solace.com/Messaging/SMF-Topics/Wildcard-Charaters-Topic-Subs.htm

## Further Reading

- [Naming Convention Guide](./references/naming-guide.md)
- [Multi-Tenant Patterns](./references/multi-tenant.md)
- [Version Migration](./references/versioning.md)
