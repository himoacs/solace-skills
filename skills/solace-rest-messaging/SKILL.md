---
name: solace-rest-messaging
description: 'Solace REST messaging gateway guidance. Use for REST Delivery Points (RDPs), MicroGateway configuration, webhook integration, HTTP content-type handling, REST consumer patterns, and OpenAPI to Solace mapping.'
argument-hint: 'rdp, webhook, microgateway, or http-publish'
---

# Solace REST Messaging

Expert guidance for REST/HTTP messaging with Solace PubSub+.

## When to Use

- **REST Publishing**: HTTP POST to Solace topics/queues
- **REST Delivery Points**: Outbound webhooks from Solace
- **Webhook Integration**: Connect external HTTP services
- **Microgateway**: API gateway functionality
- **Protocol Bridge**: Connect REST clients to SMF/MQTT

## REST Publishing (Inbound)

### Basic HTTP Publish

```bash
# Publish to topic
curl -X POST "http://broker:9000/TOPIC/acme/orders/created" \
  -H "Content-Type: application/json" \
  -H "Solace-Delivery-Mode: direct" \
  -u user:password \
  -d '{"orderId": "12345", "amount": 99.99}'

# Publish to queue (guaranteed)
curl -X POST "http://broker:9000/QUEUE/order-processor" \
  -H "Content-Type: application/json" \
  -H "Solace-Delivery-Mode: persistent" \
  -u user:password \
  -d '{"orderId": "12345"}'
```

### REST Headers

| Header | Values | Description |
|--------|--------|-------------|
| `Solace-Delivery-Mode` | `direct`, `persistent`, `non-persistent` | Message persistence |
| `Solace-Reply-Wait-Time-In-ms` | Integer | Sync request timeout |
| `Solace-Correlation-ID` | String | Message correlation |
| `Solace-Priority` | 0-255 | Message priority |
| `Solace-Time-To-Live-In-ms` | Integer | Message TTL |

### Request/Reply Pattern

```bash
# Synchronous request (waits for reply)
curl -X POST "http://broker:9000/TOPIC/services/calculator/add" \
  -H "Content-Type: application/json" \
  -H "Solace-Reply-Wait-Time-In-ms: 5000" \
  -u user:password \
  -d '{"a": 5, "b": 3}'

# Response body contains reply message
```

### Python REST Publisher

```python
import requests
from requests.auth import HTTPBasicAuth

class SolaceRestPublisher:
    def __init__(self, host: str, username: str, password: str):
        self.base_url = f"http://{host}:9000"
        self.auth = HTTPBasicAuth(username, password)
    
    def publish_topic(self, topic: str, payload: dict, 
                      delivery_mode: str = "direct") -> bool:
        url = f"{self.base_url}/TOPIC/{topic}"
        headers = {
            "Content-Type": "application/json",
            "Solace-Delivery-Mode": delivery_mode
        }
        
        response = requests.post(url, json=payload, 
                                 headers=headers, auth=self.auth)
        return response.ok
    
    def publish_queue(self, queue: str, payload: dict) -> bool:
        url = f"{self.base_url}/QUEUE/{queue}"
        headers = {
            "Content-Type": "application/json",
            "Solace-Delivery-Mode": "persistent"
        }
        
        response = requests.post(url, json=payload,
                                 headers=headers, auth=self.auth)
        return response.ok
    
    def request_reply(self, topic: str, payload: dict, 
                      timeout_ms: int = 5000) -> dict:
        url = f"{self.base_url}/TOPIC/{topic}"
        headers = {
            "Content-Type": "application/json",
            "Solace-Reply-Wait-Time-In-ms": str(timeout_ms)
        }
        
        response = requests.post(url, json=payload,
                                 headers=headers, auth=self.auth)
        response.raise_for_status()
        return response.json()

# Usage
publisher = SolaceRestPublisher("broker.example.com", "user", "pass")
publisher.publish_topic("orders/created", {"orderId": "123"})
result = publisher.request_reply("services/lookup", {"id": "123"})
```

## REST Delivery Points (Outbound)

RDPs deliver messages from Solace queues to external HTTP endpoints.

### Create RDP via SEMP

```bash
# 1. Create REST Delivery Point
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "restDeliveryPointName": "webhook-rdp",
    "enabled": true,
    "service": "webhook-queue"
  }'

# 2. Create REST Consumer
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/webhook-rdp/restConsumers" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "restConsumerName": "order-webhook",
    "enabled": true,
    "remoteHost": "api.example.com",
    "remotePort": 443,
    "tlsEnabled": true,
    "httpMethod": "post",
    "authenticationScheme": "http-basic",
    "authenticationHttpBasicUsername": "webhook-user",
    "authenticationHttpBasicPassword": "webhook-secret"
  }'

# 3. Create Queue Binding
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/webhook-rdp/queueBindings" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueBindingName": "order-events-queue",
    "postRequestTarget": "/webhooks/orders"
  }'
```

### RDP Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    Publisher    │────▶│  Solace Queue   │────▶│   REST Target   │
│  (Any Protocol) │     │  order-events   │     │ api.example.com │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                              ▼
                        ┌─────────────────┐
                        │ REST Delivery   │
                        │    Point        │
                        │                 │
                        │ - REST Consumer │
                        │ - Queue Binding │
                        └─────────────────┘
```

### Dynamic Target URL

```bash
# Use message properties in URL
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/restDeliveryPoints/webhook-rdp/queueBindings" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueBindingName": "dynamic-queue",
    "postRequestTarget": "/webhooks/${solace.user-property.tenant-id}/orders"
  }'

# Message with user property tenant-id=acme
# → POST /webhooks/acme/orders
```

## REST Consumer Configuration

### Authentication Options

```json
{
  "authenticationScheme": "http-basic",
  "authenticationHttpBasicUsername": "user",
  "authenticationHttpBasicPassword": "pass"
}

{
  "authenticationScheme": "oauth-client-credentials",
  "authenticationOauthClientId": "client-id",
  "authenticationOauthClientSecret": "secret",
  "authenticationOauthClientTokenEndpoint": "https://auth.example.com/oauth/token"
}

{
  "authenticationScheme": "oauth-jwt",
  "authenticationOauthJwtSecretKey": "jwt-signing-key"
}
```

### Retry and Failure Handling

```json
{
  "restConsumerName": "resilient-consumer",
  "maxPostWaitTime": 30,
  "retryDelay": 3,
  "outgoingConnectionCount": 5,
  "httpMethod": "post"
}
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| `maxPostWaitTime` | Response timeout (seconds) | 30 |
| `retryDelay` | Delay between retries | 3 |
| `outgoingConnectionCount` | Connection pool size | 3 |

### HTTP Response Handling

| Response Code | Solace Action |
|---------------|---------------|
| 2xx | Message acknowledged |
| 4xx | Message rejected (DMQ) |
| 5xx | Retry later |
| Timeout | Retry later |

## MicroGateway

Solace MicroGateway provides API gateway capabilities for REST services.

### Use Cases

- Rate limiting
- Request transformations
- Protocol translation
- API versioning

### Configuration

```yaml
# microgateway-config.yaml
services:
  - name: order-service
    basePath: /api/orders
    targets:
      - host: backend.example.com
        port: 8080
    
    routes:
      - path: /
        method: POST
        destination:
          type: queue
          name: orders-inbound
      
      - path: /{orderId}
        method: GET
        destination:
          type: topic
          pattern: orders/query/{orderId}
          replyTimeout: 5000
```

## OpenAPI to Solace Mapping

### AsyncAPI Generation

```yaml
# asyncapi.yaml for REST-to-Event mapping
asyncapi: '2.6.0'
info:
  title: Order Events API
  version: '1.0.0'

channels:
  orders/created:
    publish:
      operationId: orderCreated
      message:
        $ref: '#/components/messages/OrderCreatedEvent'

components:
  messages:
    OrderCreatedEvent:
      contentType: application/json
      payload:
        type: object
        properties:
          orderId:
            type: string
          amount:
            type: number
```

## Best Practices

### REST Publishing

✅ **DO**:
```bash
# Use appropriate delivery mode
-H "Solace-Delivery-Mode: persistent"  # Important messages

# Set reasonable timeouts
-H "Solace-Reply-Wait-Time-In-ms: 5000"

# Include correlation IDs
-H "Solace-Correlation-ID: req-12345"
```

❌ **DON'T**:
```bash
# Don't use persistent for high-volume telemetry
# Don't set overly long timeouts for sync requests
# Don't ignore authentication
```

### REST Delivery Points

✅ **DO**:
- Use multiple REST consumers for load balancing
- Configure appropriate retry delays
- Handle 4xx vs 5xx differently
- Monitor queue depth

❌ **DON'T**:
- Single consumer for critical workloads
- Ignore authentication security
- Set very short timeouts

### Monitoring RDP Health

```bash
# Check RDP status
curl "http://broker:8080/SEMP/v2/monitor/msgVpns/default/restDeliveryPoints/webhook-rdp/restConsumers" \
  -u admin:admin | jq '.data[] | {name, up, lastConnectionFailureReason}'

# Check queue binding status
curl "http://broker:8080/SEMP/v2/monitor/msgVpns/default/restDeliveryPoints/webhook-rdp/queueBindings" \
  -u admin:admin | jq '.data[]'
```

## Reference Links

- **REST Messaging**: https://docs.solace.com/API/RESTMessagingPrtl/Solace-REST-Overview.htm
- **REST Delivery Points**: https://docs.solace.com/Messaging/REST-Delivery-Points.htm
- **REST Consumer**: https://docs.solace.com/API/RESTMessagingPrtl/REST-Consumers.htm
- **Tutorials**: https://tutorials.solace.dev/rest

## Further Reading

- [Webhook Patterns](./references/webhooks.md)
- [Security Configuration](./references/rest-security.md)
- [Error Handling](./references/rest-errors.md)
