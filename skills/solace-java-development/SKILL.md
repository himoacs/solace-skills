---
name: solace-java-development
description: 'Solace Java API development guidance. Use for JCSMP API patterns, Solace Java API (modern), JMS 1.1/2.0 integration, JavaRTO low-latency, connection factories, JNDI configuration, transactions, error handling, and reconnection strategies.'
argument-hint: 'jcsmp, jms, javarto, transactions, or connection'
---

# Solace Java Development

Expert guidance for building Java applications with Solace PubSub+ messaging APIs.

## When to Use

- **JCSMP**: Classic high-throughput Java API
- **Solace Java API**: Modern async API with CompletableFuture
- **JMS**: Standard JMS 1.1 / 2.0 integration
- **JavaRTO**: Ultra-low-latency JNI wrapper
- **Transactions**: Local and XA transaction support
- **Connection Patterns**: Session management, reconnection

## API Comparison

| API | Latency | Throughput | Use Case |
|-----|---------|------------|----------|
| **JCSMP** | Low | High | General-purpose, most features |
| **Solace Java API** | Low | High | Modern async patterns |
| **JMS** | Medium | Medium | Standard compliance, portability |
| **JavaRTO** | Ultra-low | Very High | Trading, real-time systems |

## JCSMP API

### Maven Dependency

```xml
<dependency>
  <groupId>com.solacesystems</groupId>
  <artifactId>sol-jcsmp</artifactId>
  <version>10.23.0</version>
</dependency>
```

### Basic Connection

```java
import com.solacesystems.jcsmp.*;

public class SolaceConnection {
    
    public JCSMPSession createSession() throws JCSMPException {
        JCSMPProperties properties = new JCSMPProperties();
        properties.setProperty(JCSMPProperties.HOST, "tcp://localhost:55555");
        properties.setProperty(JCSMPProperties.VPN_NAME, "default");
        properties.setProperty(JCSMPProperties.USERNAME, "user");
        properties.setProperty(JCSMPProperties.PASSWORD, "password");
        
        // Reconnection settings
        properties.setProperty(JCSMPProperties.REAPPLY_SUBSCRIPTIONS, true);
        properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, -1); // infinite
        properties.setProperty(JCSMPProperties.RECONNECT_RETRY_WAIT_IN_MILLIS, 3000);
        
        JCSMPSession session = JCSMPFactory.onlyInstance().createSession(properties);
        session.connect();
        
        return session;
    }
}
```

### Publishing Messages

```java
public class Publisher {
    private final JCSMPSession session;
    private final XMLMessageProducer producer;
    
    public Publisher(JCSMPSession session) throws JCSMPException {
        this.session = session;
        this.producer = session.getMessageProducer(new PublishEventHandler());
    }
    
    // Direct (non-persistent) publish
    public void publishDirect(String topicName, String payload) throws JCSMPException {
        Topic topic = JCSMPFactory.onlyInstance().createTopic(topicName);
        TextMessage message = JCSMPFactory.onlyInstance().createMessage(TextMessage.class);
        message.setText(payload);
        message.setDeliveryMode(DeliveryMode.DIRECT);
        
        producer.send(message, topic);
    }
    
    // Persistent (guaranteed) publish
    public void publishPersistent(String queueName, String payload) throws JCSMPException {
        Queue queue = JCSMPFactory.onlyInstance().createQueue(queueName);
        TextMessage message = JCSMPFactory.onlyInstance().createMessage(TextMessage.class);
        message.setText(payload);
        message.setDeliveryMode(DeliveryMode.PERSISTENT);
        
        // Correlation for tracking acknowledgments
        message.setCorrelationKey(UUID.randomUUID().toString());
        
        producer.send(message, queue);
    }
    
    // Event handler for publish acknowledgments
    private static class PublishEventHandler implements JCSMPStreamingPublishCorrelatingEventHandler {
        @Override
        public void handleErrorEx(Object correlationKey, JCSMPException cause, long timestamp) {
            System.err.println("Publish failed for: " + correlationKey + ", error: " + cause);
        }
        
        @Override
        public void responseReceivedEx(Object correlationKey) {
            System.out.println("Publish acknowledged: " + correlationKey);
        }
    }
}
```

### Subscribing to Messages

```java
public class Subscriber {
    private final JCSMPSession session;
    
    // Direct subscriber (topic-based)
    public void subscribeToTopic(String topicPattern) throws JCSMPException {
        Topic topic = JCSMPFactory.onlyInstance().createTopic(topicPattern);
        
        XMLMessageConsumer consumer = session.getMessageConsumer(new MessageHandler());
        session.addSubscription(topic);
        consumer.start();
    }
    
    // Queue subscriber (guaranteed delivery)
    public void subscribeToQueue(String queueName) throws JCSMPException {
        Queue queue = JCSMPFactory.onlyInstance().createQueue(queueName);
        
        ConsumerFlowProperties flowProps = new ConsumerFlowProperties();
        flowProps.setEndpoint(queue);
        flowProps.setAckMode(JCSMPProperties.SUPPORTED_MESSAGE_ACK_CLIENT);
        
        FlowReceiver flowReceiver = session.createFlow(
            new MessageHandler(),
            flowProps,
            new EndpointProperties()
        );
        flowReceiver.start();
    }
    
    private static class MessageHandler implements XMLMessageListener {
        @Override
        public void onReceive(BytesXMLMessage message) {
            try {
                if (message instanceof TextMessage) {
                    String text = ((TextMessage) message).getText();
                    System.out.println("Received: " + text);
                }
                // Acknowledge the message
                message.ackMessage();
            } catch (Exception e) {
                // Don't ack - message will be redelivered
                System.err.println("Processing failed: " + e.getMessage());
            }
        }
        
        @Override
        public void onException(JCSMPException exception) {
            System.err.println("Consumer exception: " + exception);
        }
    }
}
```

### Request-Reply Pattern

```java
public class RequestReply {
    private final JCSMPSession session;
    private final Requestor requestor;
    
    public RequestReply(JCSMPSession session) throws JCSMPException {
        this.session = session;
        this.requestor = session.createRequestor();
    }
    
    public String sendRequest(String topic, String request, int timeoutMs) 
            throws JCSMPException {
        Topic destination = JCSMPFactory.onlyInstance().createTopic(topic);
        
        TextMessage requestMsg = JCSMPFactory.onlyInstance()
            .createMessage(TextMessage.class);
        requestMsg.setText(request);
        
        BytesXMLMessage reply = requestor.request(requestMsg, timeoutMs, destination);
        
        if (reply instanceof TextMessage) {
            return ((TextMessage) reply).getText();
        }
        return null;
    }
}
```

## Solace Java API (Modern)

### Maven Dependency

```xml
<dependency>
  <groupId>com.solace</groupId>
  <artifactId>solace-messaging-client</artifactId>
  <version>1.6.0</version>
</dependency>
```

### Modern Async Pattern

```java
import com.solace.messaging.MessagingService;
import com.solace.messaging.config.SolaceProperties.*;
import com.solace.messaging.publisher.DirectMessagePublisher;
import com.solace.messaging.receiver.DirectMessageReceiver;
import com.solace.messaging.resources.Topic;

public class ModernSolaceApp {
    
    public MessagingService connect() {
        Properties properties = new Properties();
        properties.setProperty(TransportLayerProperties.HOST, "tcp://localhost:55555");
        properties.setProperty(ServiceProperties.VPN_NAME, "default");
        properties.setProperty(AuthenticationProperties.SCHEME_BASIC_USER_NAME, "user");
        properties.setProperty(AuthenticationProperties.SCHEME_BASIC_PASSWORD, "password");
        
        MessagingService service = MessagingService.builder(ConfigurationProfile.V1)
            .fromProperties(properties)
            .build();
        
        return service.connect();
    }
    
    public void publishAsync(MessagingService service, String topicName, String payload) {
        DirectMessagePublisher publisher = service.createDirectMessagePublisherBuilder()
            .onBackPressureWait(1)
            .build()
            .start();
        
        Topic topic = Topic.of(topicName);
        OutboundMessage message = service.messageBuilder().build(payload);
        
        publisher.publish(message, topic);
    }
    
    public void subscribeAsync(MessagingService service, String topicPattern) {
        DirectMessageReceiver receiver = service.createDirectMessageReceiverBuilder()
            .withSubscriptions(TopicSubscription.of(topicPattern))
            .build()
            .start();
        
        receiver.receiveAsync(message -> {
            String payload = message.getPayloadAsString();
            System.out.println("Received: " + payload);
        });
    }
}
```

## JMS Integration

### Maven Dependencies

```xml
<dependency>
  <groupId>com.solace</groupId>
  <artifactId>sol-jms</artifactId>
  <version>10.23.0</version>
</dependency>
<dependency>
  <groupId>jakarta.jms</groupId>
  <artifactId>jakarta.jms-api</artifactId>
  <version>3.1.0</version>
</dependency>
```

### JMS Connection

```java
import com.solacesystems.jms.SolConnectionFactory;
import com.solacesystems.jms.SolJmsUtility;

public class JmsConnection {
    
    public Connection createConnection() throws JMSException {
        SolConnectionFactory factory = SolJmsUtility.createConnectionFactory();
        factory.setHost("tcp://localhost:55555");
        factory.setVPN("default");
        factory.setUsername("user");
        factory.setPassword("password");
        
        // Enable direct transport for best performance
        factory.setDirectTransport(true);
        
        Connection connection = factory.createConnection();
        connection.start();
        
        return connection;
    }
    
    public void publishJms(Connection connection, String topicName, String payload) 
            throws JMSException {
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        Topic topic = session.createTopic(topicName);
        MessageProducer producer = session.createProducer(topic);
        
        TextMessage message = session.createTextMessage(payload);
        producer.send(message);
        
        producer.close();
        session.close();
    }
}
```

## Error Handling Patterns

### Robust Session Management

```java
public class RobustSession {
    private JCSMPSession session;
    private final JCSMPProperties properties;
    private final AtomicBoolean connected = new AtomicBoolean(false);
    
    public RobustSession(JCSMPProperties properties) {
        this.properties = properties;
        
        // Configure reconnection
        properties.setProperty(JCSMPProperties.REAPPLY_SUBSCRIPTIONS, true);
        properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, -1);
        properties.setProperty(JCSMPProperties.RECONNECT_RETRY_WAIT_IN_MILLIS, 3000);
    }
    
    public void connect() throws JCSMPException {
        session = JCSMPFactory.onlyInstance().createSession(properties, null, 
            new SessionEventHandler() {
                @Override
                public void handleEvent(SessionEventArgs event) {
                    switch (event.getEvent()) {
                        case RECONNECTING:
                            connected.set(false);
                            System.out.println("Reconnecting...");
                            break;
                        case RECONNECTED:
                            connected.set(true);
                            System.out.println("Reconnected!");
                            break;
                        case DOWN_ERROR:
                            connected.set(false);
                            System.err.println("Session down: " + event.getInfo());
                            break;
                    }
                }
            }
        );
        session.connect();
        connected.set(true);
    }
    
    public boolean isConnected() {
        return connected.get();
    }
}
```

### Consumer with Error Handling

```java
public class RobustConsumer implements XMLMessageListener {
    private final Queue dlq;
    private final XMLMessageProducer dlqProducer;
    private final int maxRetries = 3;
    
    @Override
    public void onReceive(BytesXMLMessage message) {
        int retryCount = getRetryCount(message);
        
        try {
            processMessage(message);
            message.ackMessage();
        } catch (RecoverableException e) {
            if (retryCount < maxRetries) {
                // Will be redelivered
                System.out.println("Recoverable error, will retry: " + e.getMessage());
            } else {
                sendToDlq(message, e);
                message.ackMessage();
            }
        } catch (Exception e) {
            // Unrecoverable - send to DLQ immediately
            sendToDlq(message, e);
            message.ackMessage();
        }
    }
    
    private void sendToDlq(BytesXMLMessage original, Exception error) {
        try {
            // Create new message for DLQ with error info
            BytesXMLMessage dlqMessage = JCSMPFactory.onlyInstance()
                .createMessage(BytesXMLMessage.class);
            dlqMessage.writeBytes(original.getBytes());
            dlqMessage.setUserPropertyValue("error_message", error.getMessage());
            dlqMessage.setUserPropertyValue("original_destination", 
                original.getDestination().getName());
            
            dlqProducer.send(dlqMessage, dlq);
        } catch (JCSMPException e) {
            System.err.println("Failed to send to DLQ: " + e);
        }
    }
}
```

## Best Practices

### Connection Management

```java
// Use try-with-resources or proper cleanup
try (JCSMPSession session = createSession()) {
    // Use session
} catch (JCSMPException e) {
    // Handle error
}

// Or with shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    if (session != null && !session.isClosed()) {
        session.closeSession();
    }
}));
```

### Thread Safety

```java
// Sessions are NOT thread-safe for publishing
// Option 1: Synchronize
synchronized (producer) {
    producer.send(message, destination);
}

// Option 2: Thread-local producers (preferred for high throughput)
private final ThreadLocal<XMLMessageProducer> producerLocal = 
    ThreadLocal.withInitial(() -> createProducer());
```

### Performance Tuning

```java
// Publisher settings
properties.setProperty(JCSMPProperties.PUB_ACK_WINDOW_SIZE, 255);

// Subscriber settings  
flowProps.setWindowSize(255);
flowProps.setMaxUnackedMessages(1000);

// Transport optimization
properties.setProperty(JCSMPProperties.SOCKET_SEND_BUFFER_SIZE, 65536);
properties.setProperty(JCSMPProperties.SOCKET_RECEIVE_BUFFER_SIZE, 65536);
```

## Official JCSMP Best Practices (from Solace Documentation)

### Threading Model Selection

```java
// RECOMMENDATION: Use 'one session, one context' model for most cases
// Consider dispatching from I/O thread for ultra-low latency

JCSMPProperties properties = new JCSMPProperties();

// For ultra-low latency (dispatch directly from reactor thread)
// WARNING: Do NOT block in callbacks when using this setting
properties.setBooleanProperty(JCSMPProperties.MESSAGE_CALLBACK_ON_REACTOR, true);

// Trade-off: Reduces latency but also reduces max throughput
// Messages are processed individually instead of in batches
```

### Session Establishment Best Practices

```java
// HA Failover: Configure reconnect duration for at least 5 minutes
// HA failovers typically complete within 30 seconds, but allow buffer
properties.setProperty(JCSMPProperties.CONNECT_RETRIES, 1);
properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, 20);
properties.setProperty(JCSMPProperties.RECONNECT_RETRY_WAIT_IN_MILLIS, 3000);
properties.setProperty(JCSMPProperties.CONNECT_RETRIES_PER_HOST, 5);

// For Replication Failover: Use infinite retries
// Replication failover duration is non-deterministic (may require operator intervention)
properties.setProperty(JCSMPProperties.RECONNECT_RETRIES, -1);

// Host lists for software event broker HA
// Order matters: primary first, then backup
properties.setProperty(JCSMPProperties.HOST, "tcp://primary:55555,tcp://backup:55555");

// Reapply subscriptions on reconnect (RECOMMENDED)
properties.setProperty(JCSMPProperties.REAPPLY_SUBSCRIPTIONS, true);
```

### Keep-Alive Alignment

```java
// CRITICAL: Client keep-alive should be same order of magnitude as broker TCP keep-alive
// Default broker: 3s idle + 5 probes at 1s = 8s to detect failure
// Default API: 3s interval, 3 missed = 9s to detect failure

// If API keep-alive is too aggressive, connection may drop prematurely
// BAD: 500ms interval with 3 retries = premature disconnection
properties.setProperty(JCSMPProperties.KEEP_ALIVE_INTERVAL, 500); // TOO AGGRESSIVE

// GOOD: Keep defaults or match broker settings
properties.setProperty(JCSMPProperties.KEEP_ALIVE_INTERVAL, 3000);
properties.setProperty(JCSMPProperties.KEEP_ALIVE_LIMIT, 3);
```

### TCP Buffer Sizing for Large Messages

```java
// For large messages or WAN optimization, increase buffer sizes
// Default Java socket buffer: 64KB
JCSMPChannelProperties channelProps = new JCSMPChannelProperties();
channelProps.setSendBuffer(131072);     // 128KB send buffer
channelProps.setReceiveBuffer(262144);  // 256KB receive buffer

// Windows: Receive buffer should be larger than send buffer (ratio 5:3)
// channelProps.setSendBuffer(90000);
// channelProps.setReceiveBuffer(150000);

properties.setProperty(JCSMPProperties.CLIENT_CHANNEL_PROPERTIES, channelProps);

// Also configure Linux TCP buffers if needed:
// /etc/sysctl.conf:
// net.core.rmem_max = 16777216
// net.core.wmem_max = 16777216
// net.ipv4.tcp_rmem = 4096 87380 16777216
// net.ipv4.tcp_wmem = 4096 65536 16777216
```

### Message Sending Best Practices

```java
// USE SESSION-INDEPENDENT MESSAGES (recommended for new applications)
// No performance penalty if messages are pre-allocated and reused

// Pre-allocate and reuse for best performance
private final TextMessage reusableMessage = 
    JCSMPFactory.onlyInstance().createMessage(TextMessage.class);

public void sendEfficiently(String payload, Topic topic) throws JCSMPException {
    reusableMessage.reset();  // Clear previous content
    reusableMessage.setText(payload);
    producer.send(reusableMessage, topic);
}

// BATCH SENDING: Send up to 50 messages in one API call
TextMessage[] batch = new TextMessage[50];
for (int i = 0; i < 50; i++) {
    batch[i] = createMessage(i);
}
producer.sendMultiple(batch, topic);  // Much faster than 50 individual sends

// TTL: Set TTL on guaranteed messages to prevent queue buildup
message.setTimeToLive(60000);  // 60 seconds
// Also enable respect-ttl on the queue via broker config
```

### Message Receiving Best Practices

```java
// CRITICAL: Return from callbacks as quickly as possible
// Do NOT block in message callbacks

public void onReceive(BytesXMLMessage message) {
    // BAD: Blocking operations in callback
    // httpClient.post(message.getText());  // BLOCKS!
    // database.insert(message);            // BLOCKS!
    
    // GOOD: Hand off to worker thread pool
    executor.submit(() -> {
        processMessage(message);
    });
    
    // Acknowledge after queuing (if using CLIENT_ACK)
    message.ackMessage();
}

// Handle redelivered messages (flag persists during HA failover)
if (message.getRedelivered()) {
    // Message was previously delivered - handle idempotently
    // Note: Flag does NOT persist during DR replication
}

// Handle unexpected message formats gracefully
// ALWAYS acknowledge to prevent infinite redelivery loops
try {
    processMessage(message);
} catch (Exception e) {
    log.error("Failed to process message", e);
    message.ackMessage();  // Ack even on failure to prevent loops
}
```

### Flow Configuration (Guaranteed Messaging)

```java
ConsumerFlowProperties flowProps = new ConsumerFlowProperties();
flowProps.setEndpoint(queue);
flowProps.setAckMode(JCSMPProperties.SUPPORTED_MESSAGE_ACK_CLIENT);

// AD Window Size should NOT exceed queue's max-delivered-unacked-msgs-per-flow
// If AD window > max-unacked, broker is limited to max-unacked anyway
flowProps.setTransportWindowSize(25);  // Match queue setting

// For single-message processing (strict ordering):
// Set both to 1 for consistent delivery timing
flowProps.setTransportWindowSize(1);
// Configure queue: max-delivered-unacked-msgs-per-flow = 1

// Size flows to stay within broker's G-1 queue work units (default: 20,000)
// Work unit = 2048 bytes
// Example: 10 flows × 255 window × 20KB avg = 24,902 work units (EXCEEDS LIMIT)
// Better:  10 flows × 25 window × 20KB avg = 2,441 work units (OK)
```

### Always Clean Up Resources

```java
// JCSMPSession, XMLMessageProducer, XMLMessageConsumer allocate system resources
// Close them properly when no longer needed

// Closing JCSMPSession also closes associated Producer and Consumer
try {
    if (consumer != null) consumer.close();
    if (producer != null) producer.close();
    if (session != null && !session.isClosed()) session.closeSession();
} catch (JCSMPException e) {
    log.error("Error during cleanup", e);
}

// Or use shutdown hook for graceful shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    cleanup();
}));

// DO NOT cache XMLMessages from producer message pool
// API may automatically recycle them
```

### Error Handling Events

```java
// Register session event handler for all sessions
session.addSessionEventHandler(new SessionEventHandler() {
    @Override
    public void handleEvent(SessionEventArgs event) {
        switch (event.getEvent()) {
            case RECONNECTING:
                log.warn("Connection lost, reconnecting...");
                break;
            case RECONNECTED:
                log.info("Reconnected successfully");
                break;
            case DOWN_ERROR:
                log.error("Session down: " + event.getInfo());
                break;
            case SUBSCRIPTION_ERROR:
                log.error("Subscription rejected: " + event.getReason());
                break;
        }
    }
});

// Register JCSMPReconnectEventHandler for transport events (even publish-only apps)
XMLMessageConsumer consumer = session.getMessageConsumer(
    new JCSMPReconnectEventHandler() {
        @Override
        public boolean preReconnect() {
            log.info("About to reconnect...");
            return true;  // Continue reconnect
        }
        
        @Override
        public void postReconnect() {
            log.info("Reconnected, re-establishing state...");
        }
    },
    messageListener
);
// For publish-only apps: create empty consumer (don't need to start it)
```

## Reference Links

- **JCSMP API Documentation**: https://docs.solace.com/API/Messaging-APIs/JCSMP-API/jcsmp-api-home.htm
- **Solace Java API**: https://docs.solace.com/API/Messaging-APIs/Java-API/java-api-home.htm
- **JMS Guide**: https://docs.solace.com/API/Solace-JMS-API/jms-get-started-open.htm
- **Tutorials**: https://tutorials.solace.dev/jcsmp
- **GitHub Samples**: https://github.com/SolaceSamples/solace-samples-java-jcsmp

## Further Reading

- [JCSMP Patterns](./references/jcsmp-patterns.md)
- [JMS Integration](./references/jms-integration.md)
- [Transactions](./references/transactions.md)
- [Performance Tuning](./references/performance.md)
