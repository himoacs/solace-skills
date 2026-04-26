---
name: solace-spring-development
description: 'Solace Spring ecosystem guidance. Use for Spring Cloud Stream with Solace Binder, Spring Boot auto-configuration, reactive streams, Spring Integration patterns, configuration properties, and testing with embedded broker.'
argument-hint: 'spring-cloud-stream, boot, reactive, or integration'
---

# Solace Spring Development

Expert guidance for integrating Solace PubSub+ with Spring Framework applications.

## When to Use

- **Spring Cloud Stream**: Building event-driven microservices
- **Spring Boot**: Auto-configuration and starter dependencies
- **Reactive**: Reactive streams with Project Reactor
- **Spring Integration**: Enterprise integration patterns
- **Testing**: Unit and integration testing with Solace

## Spring Cloud Stream with Solace Binder

### Maven Dependencies

```xml
<dependency>
    <groupId>com.solace.spring.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-solace</artifactId>
    <version>4.3.0</version>
</dependency>
```

### Application Configuration

```yaml
# application.yml
spring:
  cloud:
    stream:
      # Binder configuration
      binders:
        solace:
          type: solace
          environment:
            solace:
              java:
                host: tcp://localhost:55555
                msgVpn: default
                clientUsername: user
                clientPassword: password
      
      # Binding configuration
      bindings:
        # Consumer binding
        orderInput-in-0:
          destination: orders/created
          binder: solace
          group: order-processor
        
        # Producer binding
        orderOutput-out-0:
          destination: orders/processed
          binder: solace
      
      # Solace-specific binding properties
      solace:
        bindings:
          orderInput-in-0:
            consumer:
              queueName: order-processor-queue
              queueAccessType: exclusive
              queueMaxMsgRedelivery: 3
          orderOutput-out-0:
            producer:
              destination-type: topic
```

### Functional Consumer

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.function.Consumer;

@Configuration
public class OrderConsumerConfig {
    
    @Bean
    public Consumer<Order> orderInput() {
        return order -> {
            System.out.println("Processing order: " + order.getId());
            // Process the order
        };
    }
}
```

### Functional Producer

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.function.Supplier;

@Configuration
public class OrderProducerConfig {
    
    @Bean
    public Supplier<Order> orderOutput() {
        return () -> {
            // Generate or fetch order
            return new Order(UUID.randomUUID().toString(), "NEW");
        };
    }
}
```

### Functional Processor (Stream)

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.function.Function;

@Configuration
public class OrderProcessorConfig {
    
    @Bean
    public Function<Order, ProcessedOrder> processOrder() {
        return order -> {
            // Transform order
            return ProcessedOrder.from(order)
                .withStatus("PROCESSED")
                .withProcessedAt(Instant.now());
        };
    }
}
```

### Manual Message Handling

```java
import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

@Service
public class OrderService {
    
    private final StreamBridge streamBridge;
    
    public OrderService(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }
    
    public void publishOrder(Order order) {
        Message<Order> message = MessageBuilder.withPayload(order)
            .setHeader("orderType", order.getType())
            .setHeader("priority", order.getPriority())
            .build();
        
        streamBridge.send("orderOutput-out-0", message);
    }
    
    // Publish to dynamic destination
    public void publishToTopic(String topic, Object payload) {
        streamBridge.send(topic, payload);
    }
}
```

## Spring Boot Auto-Configuration

### Starter Dependency

```xml
<dependency>
    <groupId>com.solace.spring.boot</groupId>
    <artifactId>solace-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

### Configuration Properties

```yaml
# application.yml
solace:
  java:
    host: tcp://localhost:55555
    msgVpn: default
    clientUsername: user
    clientPassword: password
    
    # Connection properties
    reconnectRetries: -1
    reconnectRetryWaitInMillis: 3000
    connectRetriesPerHost: 5
    
    # Advanced settings
    apiProperties:
      SUB_ACK_WINDOW_SIZE: 255
      PUB_ACK_WINDOW_SIZE: 255
```

### Auto-wired Session

```java
import com.solacesystems.jcsmp.JCSMPSession;
import org.springframework.beans.factory.annotation.Autowired;

@Service
public class SolaceService {
    
    @Autowired
    private JCSMPSession jcsmpSession;
    
    public void sendMessage(String topic, String payload) throws JCSMPException {
        XMLMessageProducer producer = jcsmpSession.getMessageProducer(
            new JCSMPStreamingPublishCorrelatingEventHandler() {
                // ... handlers
            }
        );
        
        TextMessage message = JCSMPFactory.onlyInstance()
            .createMessage(TextMessage.class);
        message.setText(payload);
        
        producer.send(message, JCSMPFactory.onlyInstance().createTopic(topic));
    }
}
```

## Reactive Streams

### Reactive Consumer

```java
import reactor.core.publisher.Flux;
import org.springframework.context.annotation.Bean;

@Configuration
public class ReactiveConfig {
    
    @Bean
    public Consumer<Flux<Order>> reactiveOrderInput() {
        return orderFlux -> orderFlux
            .doOnNext(order -> log.info("Received: {}", order))
            .flatMap(this::processAsync)
            .subscribe();
    }
    
    private Mono<ProcessedOrder> processAsync(Order order) {
        return Mono.fromCallable(() -> processOrder(order))
            .subscribeOn(Schedulers.boundedElastic());
    }
}
```

### Reactive Producer

```java
@Configuration
public class ReactiveProducerConfig {
    
    @Bean
    public Supplier<Flux<Order>> reactiveOrderOutput() {
        return () -> Flux.interval(Duration.ofSeconds(1))
            .map(i -> new Order("order-" + i, "NEW"));
    }
}
```

## Error Handling

### Consumer Error Handler

```yaml
spring:
  cloud:
    stream:
      bindings:
        orderInput-in-0:
          consumer:
            maxAttempts: 3
            backOffInitialInterval: 1000
            backOffMultiplier: 2
      
      solace:
        bindings:
          orderInput-in-0:
            consumer:
              # Auto-bind error queue
              autoBindErrorQueue: true
              errorQueueNameOverride: order-errors
```

### Custom Error Handler

```java
@Configuration
public class ErrorHandlingConfig {
    
    @Bean
    public Consumer<ErrorMessage> errorHandler() {
        return errorMessage -> {
            Throwable error = errorMessage.getPayload();
            Message<?> originalMessage = errorMessage.getOriginalMessage();
            
            log.error("Processing failed for message: {}", 
                originalMessage.getPayload(), error);
            
            // Send to DLQ, alert, etc.
        };
    }
    
    @ServiceActivator(inputChannel = "errorChannel")
    public void handleError(ErrorMessage errorMessage) {
        // Global error handler
    }
}
```

## Testing

### Test Configuration

```java
@SpringBootTest
@Import(SolaceTestConfig.class)
class OrderProcessorTest {
    
    @Autowired
    private InputDestination input;
    
    @Autowired
    private OutputDestination output;
    
    @Test
    void shouldProcessOrder() {
        Order order = new Order("123", "NEW");
        
        input.send(new GenericMessage<>(order), "orderInput-in-0");
        
        Message<byte[]> result = output.receive(1000, "orderOutput-out-0");
        
        assertThat(result).isNotNull();
        ProcessedOrder processed = deserialize(result.getPayload());
        assertThat(processed.getStatus()).isEqualTo("PROCESSED");
    }
}

@TestConfiguration
class SolaceTestConfig {
    @Bean
    public TestChannelBinderConfiguration<?> testChannelBinder() {
        return new TestChannelBinderConfiguration<>();
    }
}
```

### Integration Test with TestContainers

```java
@SpringBootTest
@Testcontainers
class SolaceIntegrationTest {
    
    @Container
    static GenericContainer<?> solace = new GenericContainer<>("solace/solace-pubsub-standard:latest")
        .withExposedPorts(55555, 8080)
        .withEnv("username_admin_globalaccesslevel", "admin")
        .withEnv("username_admin_password", "admin");
    
    @DynamicPropertySource
    static void solaceProperties(DynamicPropertyRegistry registry) {
        registry.add("solace.java.host", 
            () -> "tcp://" + solace.getHost() + ":" + solace.getMappedPort(55555));
    }
    
    @Test
    void shouldConnectToSolace() {
        // Test with real broker
    }
}
```

## Configuration Reference

### Consumer Properties

```yaml
spring.cloud.stream.solace.bindings.<binding-name>.consumer:
  # Queue configuration
  queueName: my-queue
  queueAccessType: exclusive  # or non-exclusive
  queueMaxMsgRedelivery: 3
  queueRespectsMsgTtl: true
  
  # Error handling
  autoBindErrorQueue: true
  errorQueueNameOverride: my-error-queue
  
  # Flow control
  flowPreserveMsgOrder: true
  flowRebindWaitTimeout: 10000
```

### Producer Properties

```yaml
spring.cloud.stream.solace.bindings.<binding-name>.producer:
  # Destination type
  destination-type: topic  # or queue
  
  # Delivery mode
  deliveryMode: PERSISTENT  # or DIRECT
  
  # Headers to exclude
  headerExclusions: [internal-header]
```

## Best Practices

### Naming Conventions

```yaml
# Use descriptive binding names
spring.cloud.stream.bindings:
  orderCreatedInput-in-0:    # <event><direction>-<index>
  orderProcessedOutput-out-0:
```

### Health Indicators

```java
@Component
public class SolaceHealthIndicator implements HealthIndicator {
    
    @Autowired
    private JCSMPSession session;
    
    @Override
    public Health health() {
        if (session != null && !session.isClosed()) {
            return Health.up()
                .withDetail("vpn", session.getProperty(JCSMPProperties.VPN_NAME))
                .build();
        }
        return Health.down().build();
    }
}
```

## Reference Links

- **Spring Cloud Stream Solace Binder**: https://github.com/SolaceProducts/solace-spring-cloud
- **Spring Boot Starter**: https://github.com/SolaceProducts/solace-spring-boot
- **Tutorials**: https://tutorials.solace.dev/spring
- **Codelab**: https://codelabs.solace.dev/codelabs/solace-workshop-scs-pcf/ (175 min)

## Further Reading

- [Spring Cloud Stream Patterns](./references/spring-cloud-stream.md)
- [Configuration Reference](./references/configuration.md)
- [Testing Strategies](./references/testing.md)
