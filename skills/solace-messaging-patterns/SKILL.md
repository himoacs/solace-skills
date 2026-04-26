---
name: solace-messaging-patterns
description: 'Solace messaging pattern implementation guidance. Use for request/reply, pub/sub fan-out, competing consumers, event sourcing, saga orchestration, dead letter queue handling, content-based routing, and message replay.'
argument-hint: 'request-reply, fan-out, saga, dlq, or replay'
---

# Solace Messaging Patterns

Expert guidance for implementing messaging patterns with Solace PubSub+.

## When to Use

- **Integration Patterns**: Request/reply, pub/sub
- **Microservices**: Saga, event sourcing
- **Reliability**: DLQ, replay, ordering
- **Scalability**: Fan-out, competing consumers

## Request/Reply Pattern

### Synchronous Request/Reply

```java
// Responder
class Responder {
    private MessagingService messagingService;
    private DirectMessageReceiver receiver;
    private DirectMessagePublisher publisher;

    public void start() {
        receiver = messagingService.createDirectMessageReceiverBuilder()
            .withSubscriptions(TopicSubscription.of("services/calculator/*"))
            .build()
            .start();

        publisher = messagingService.createDirectMessagePublisherBuilder()
            .build()
            .start();

        receiver.receiveAsync(this::handleRequest);
    }

    private void handleRequest(InboundMessage request) {
        // Get reply-to destination
        Topic replyTo = request.getReplyTo();
        String correlationId = request.getCorrelationId();

        // Process request
        String payload = request.getPayloadAsString();
        String response = processRequest(payload);

        // Send reply
        OutboundMessage reply = messagingService.messageBuilder()
            .withCorrelationId(correlationId)
            .build(response);

        publisher.publish(reply, replyTo);
    }
}

// Requester
class Requester {
    public String sendRequest(String request, int timeoutMs) {
        // Create temporary reply topic
        String correlationId = UUID.randomUUID().toString();
        Topic replyTo = Topic.of("replies/" + correlationId);

        // Subscribe to reply
        DirectMessageReceiver replyReceiver = messagingService
            .createDirectMessageReceiverBuilder()
            .withSubscriptions(TopicSubscription.of(replyTo.getName()))
            .build()
            .start();

        // Send request with reply-to
        OutboundMessage requestMsg = messagingService.messageBuilder()
            .withCorrelationId(correlationId)
            .withReplyTo(replyTo)
            .build(request);

        publisher.publish(requestMsg, Topic.of("services/calculator/add"));

        // Wait for reply
        InboundMessage reply = replyReceiver.receiveMessage(timeoutMs);
        replyReceiver.terminate(1000);

        return reply.getPayloadAsString();
    }
}
```

### Asynchronous Request/Reply

```java
class AsyncRequester {
    private Map<String, CompletableFuture<String>> pendingRequests = new ConcurrentHashMap<>();

    public CompletableFuture<String> sendRequestAsync(String request) {
        String correlationId = UUID.randomUUID().toString();
        CompletableFuture<String> future = new CompletableFuture<>();
        pendingRequests.put(correlationId, future);

        OutboundMessage requestMsg = messagingService.messageBuilder()
            .withCorrelationId(correlationId)
            .withReplyTo(Topic.of("replies/" + clientId))
            .build(request);

        publisher.publish(requestMsg, Topic.of("services/calculator/add"));

        // Timeout handling
        CompletableFuture.delayedExecutor(5, TimeUnit.SECONDS)
            .execute(() -> {
                if (pendingRequests.remove(correlationId) != null) {
                    future.completeExceptionally(new TimeoutException());
                }
            });

        return future;
    }

    private void handleReply(InboundMessage reply) {
        String correlationId = reply.getCorrelationId();
        CompletableFuture<String> future = pendingRequests.remove(correlationId);
        if (future != null) {
            future.complete(reply.getPayloadAsString());
        }
    }
}
```

## Pub/Sub Fan-Out

### One-to-Many Distribution

```
Publisher ──▶ Topic: orders/created/v1 ──┬──▶ Inventory Service
                                         ├──▶ Notification Service
                                         ├──▶ Analytics Service
                                         └──▶ Audit Service
```

```java
// Publisher (single)
publisher.publish(orderCreatedEvent, Topic.of("orders/created/v1"));

// Subscribers (multiple independent consumers)
// Each subscribes to same topic
receiver.addSubscription(TopicSubscription.of("orders/created/v1"));
```

### Selective Fan-Out with Topics

```
orders/created/region/US ──┬──▶ US Regional Handler
                           └──▶ Global Analytics

orders/created/region/EU ──┬──▶ EU Regional Handler
                           └──▶ Global Analytics

# Global Analytics subscribes to: orders/created/region/*
```

## Competing Consumers

### Load-Balanced Processing

```
                        ┌──▶ Consumer 1 ──┐
Publisher ──▶ Queue ────┼──▶ Consumer 2 ──┼──▶ (each message goes to ONE consumer)
                        └──▶ Consumer 3 ──┘
```

```java
// All consumers bind to same exclusive queue
PersistentMessageReceiver consumer = messagingService
    .createPersistentMessageReceiverBuilder()
    .build(Queue.durableExclusiveQueue("order-processing-queue"));

consumer.start();
consumer.receiveAsync(this::processOrder);
```

### Queue Configuration

```bash
# Create non-exclusive queue for competing consumers
curl -X POST "$SEMP_URL/config/msgVpns/default/queues" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "order-processing-queue",
    "accessType": "non-exclusive",
    "maxMsgSpoolUsage": 5000,
    "permission": "consume"
  }'

# Add topic subscription
curl -X POST "$SEMP_URL/config/msgVpns/default/queues/order-processing-queue/subscriptions" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{"subscriptionTopic": "orders/>"}'
```

## Event Sourcing

### Event Store Pattern

```java
class EventStore {
    private PersistentMessagePublisher publisher;

    public void append(String streamId, DomainEvent event) {
        String topic = String.format("events/%s/%s/v%d",
            event.getType(),
            streamId,
            event.getVersion()
        );

        OutboundMessage message = messagingService.messageBuilder()
            .withProperty("eventType", event.getType())
            .withProperty("aggregateId", streamId)
            .withProperty("version", event.getVersion())
            .withProperty("timestamp", Instant.now().toString())
            .build(serialize(event));

        // Publish to topic (fan-out to projections)
        publisher.publish(message, Topic.of(topic));

        // Also send to ordered queue for replay
        publisher.publish(message, Queue.durableExclusiveQueue("event-store-" + streamId));
    }
}

// Projection (subscriber)
class OrderProjection {
    public void start() {
        receiver.addSubscription(TopicSubscription.of("events/OrderCreated/>"));
        receiver.addSubscription(TopicSubscription.of("events/OrderUpdated/>"));
        receiver.addSubscription(TopicSubscription.of("events/OrderCancelled/>"));

        receiver.receiveAsync(this::handleEvent);
    }

    private void handleEvent(InboundMessage message) {
        DomainEvent event = deserialize(message);
        updateReadModel(event);
    }
}
```

## Saga Pattern

### Orchestration-Based Saga

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Order     │───▶│  Inventory  │───▶│  Payment    │
│   Created   │    │  Reserved   │    │  Processed  │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
    Rollback          Rollback           Rollback
    (if fail)         (if fail)          (if fail)
```

```java
class SagaOrchestrator {
    private Map<String, SagaState> sagas = new ConcurrentHashMap<>();

    public void startSaga(CreateOrderCommand cmd) {
        String sagaId = UUID.randomUUID().toString();
        SagaState state = new SagaState(sagaId, cmd);
        sagas.put(sagaId, state);

        // Step 1: Reserve inventory
        publisher.publish(
            createCommand("ReserveInventory", cmd.getItems(), sagaId),
            Topic.of("saga/inventory/reserve")
        );
    }

    public void handleReply(InboundMessage reply) {
        String sagaId = reply.getCorrelationId();
        SagaState state = sagas.get(sagaId);
        String replyType = reply.getProperty("type");

        switch (replyType) {
            case "InventoryReserved":
                state.inventoryReserved = true;
                // Step 2: Process payment
                publisher.publish(
                    createCommand("ProcessPayment", state.getPaymentInfo(), sagaId),
                    Topic.of("saga/payment/process")
                );
                break;

            case "PaymentProcessed":
                state.paymentProcessed = true;
                // Saga complete!
                completeSaga(state);
                break;

            case "InventoryReservationFailed":
            case "PaymentFailed":
                // Rollback
                compensate(state);
                break;
        }
    }

    private void compensate(SagaState state) {
        if (state.inventoryReserved) {
            publisher.publish(
                createCommand("ReleaseInventory", state.getItems(), state.sagaId),
                Topic.of("saga/inventory/release")
            );
        }
        // Mark saga as failed
        state.status = SagaStatus.COMPENSATED;
        sagas.remove(state.sagaId);
    }
}
```

## Dead Letter Queue (DLQ)

### Configuration

```bash
# Create DLQ
curl -X POST "$SEMP_URL/config/msgVpns/default/queues" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "#dlq",
    "accessType": "exclusive",
    "maxMsgSpoolUsage": 10000
  }'

# Configure queue to use DLQ
curl -X PATCH "$SEMP_URL/config/msgVpns/default/queues/order-queue" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "rejectMsgToSenderOnDiscardBehavior": "when-queue-enabled",
    "redeliveryEnabled": true,
    "maxRedeliveryCount": 3,
    "deadMsgQueue": "#dlq"
  }'
```

### DLQ Processor

```java
class DlqProcessor {
    public void start() {
        PersistentMessageReceiver dlqReceiver = messagingService
            .createPersistentMessageReceiverBuilder()
            .build(Queue.durableExclusiveQueue("#dlq"));

        dlqReceiver.start();
        dlqReceiver.receiveAsync(this::handleDeadLetter);
    }

    private void handleDeadLetter(InboundMessage message) {
        String originalTopic = message.getProperty("JMS_Solace_DeadMsgQueueEligibleReason");
        int redeliveryCount = message.getRedeliveryCount();

        // Log for investigation
        log.error("Dead letter: topic={}, reason={}, payload={}",
            originalTopic, redeliveryCount, message.getPayloadAsString());

        // Store for manual review
        alertService.notifyDeadLetter(message);

        // Acknowledge to prevent infinite loop
        message.ack();
    }
}
```

## Message Replay

### Replay from Queue

```java
class MessageReplayer {
    public void replayMessages(String queueName, Instant fromTime) {
        // Create replay configuration
        ReplayStrategy strategy = ReplayStrategy.timeBased(fromTime);

        PersistentMessageReceiver receiver = messagingService
            .createPersistentMessageReceiverBuilder()
            .withMessageReplay(strategy)
            .build(Queue.durableExclusiveQueue(queueName));

        receiver.start();

        // Process replayed messages
        while (true) {
            InboundMessage message = receiver.receiveMessage(5000);
            if (message == null) break;

            processReplayedMessage(message);
            message.ack();
        }

        receiver.terminate(5000);
    }
}
```

### Configure Replay Log

```bash
# Enable replay on queue
curl -X PATCH "$SEMP_URL/config/msgVpns/default/queues/order-queue" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "replayEnabled": true,
    "replayMaxMsgSpoolUsage": 5000
  }'
```

## Content-Based Routing

### Topic-Based Routing

```java
// Publisher includes routing attributes in topic
String topic = String.format("orders/%s/%s/%s",
    order.getRegion(),      // orders/US/...
    order.getPriority(),    // orders/US/HIGH/...
    order.getType()         // orders/US/HIGH/ONLINE
);
publisher.publish(order, Topic.of(topic));

// Consumers subscribe to relevant patterns
highPriorityService.subscribe("orders/*/HIGH/>");
usRegionService.subscribe("orders/US/>");
```

### Selector-Based Routing

```java
// Create queue with selector
curl -X POST "$SEMP_URL/config/msgVpns/default/queues" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "queueName": "high-value-orders",
    "selector": "amount > 1000"
  }'

// Publisher sets selector property
OutboundMessage message = messagingService.messageBuilder()
    .withProperty("amount", order.getAmount())
    .build(serialize(order));
```

## Reference Links

- **Messaging Patterns**: https://docs.solace.com/Messaging/Messaging-Patterns.htm
- **Guaranteed Messaging**: https://docs.solace.com/Messaging/Guaranteed-Messaging.htm
- **Request/Reply**: https://docs.solace.com/Messaging/Request-Reply.htm

## Further Reading

- [Saga Implementation](./references/saga-patterns.md)
- [Event Sourcing](./references/event-sourcing.md)
- [Ordering Guarantees](./references/message-ordering.md)
