---
name: solace-go-development
description: 'Solace Go API development guidance. Use for solace-go-client native API, goroutine patterns for pub/sub, context cancellation, error handling idioms, and Kubernetes sidecar patterns.'
argument-hint: 'publish, subscribe, context, errors, or kubernetes'
---

# Solace Go Development

Expert guidance for building Go applications with Solace PubSub+ messaging.

## When to Use

- **Cloud-Native Apps**: Go microservices with messaging
- **Kubernetes Workloads**: Sidecar and service patterns
- **High Concurrency**: Goroutine-based pub/sub
- **Context Handling**: Proper cancellation and timeouts

## Installation

```bash
go get solace.dev/go/messaging
```

## Basic Connection

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"

    "solace.dev/go/messaging"
    "solace.dev/go/messaging/pkg/solace"
    "solace.dev/go/messaging/pkg/solace/config"
)

func main() {
    // Build messaging service
    messagingService, err := messaging.NewMessagingServiceBuilder().
        FromConfigurationProvider(config.ServicePropertyMap{
            config.TransportLayerPropertyHost:                "tcp://localhost:55555",
            config.ServicePropertyVPNName:                    "default",
            config.AuthenticationPropertySchemeBasicUserName: "user",
            config.AuthenticationPropertySchemeBasicPassword: "password",
        }).
        WithReconnectionRetryStrategy(config.RetryStrategyParameterizedRetry(20, 3000)).
        Build()

    if err != nil {
        log.Fatalf("Error building service: %v", err)
    }

    // Connect with context
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := messagingService.ConnectWithContext(ctx); err != nil {
        log.Fatalf("Error connecting: %v", err)
    }

    log.Println("Connected to Solace")

    // Graceful shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    <-sigChan

    messagingService.Disconnect()
    log.Println("Disconnected")
}
```

## Publishing

### Direct Publisher

```go
package publisher

import (
    "solace.dev/go/messaging"
    "solace.dev/go/messaging/pkg/solace"
    "solace.dev/go/messaging/pkg/solace/message"
    "solace.dev/go/messaging/pkg/solace/resource"
)

type DirectPublisher struct {
    service   solace.MessagingService
    publisher solace.DirectMessagePublisher
}

func NewDirectPublisher(service solace.MessagingService) (*DirectPublisher, error) {
    publisher, err := service.CreateDirectMessagePublisherBuilder().
        OnBackPressureWait(100). // Buffer up to 100 messages
        Build()
    if err != nil {
        return nil, err
    }

    if err := publisher.Start(); err != nil {
        return nil, err
    }

    return &DirectPublisher{
        service:   service,
        publisher: publisher,
    }, nil
}

func (p *DirectPublisher) Publish(topic string, payload []byte) error {
    msg, err := p.service.MessageBuilder().
        WithProperty("timestamp", time.Now().Unix()).
        BuildWithByteArrayPayload(payload)
    if err != nil {
        return err
    }

    return p.publisher.Publish(msg, resource.TopicOf(topic))
}

func (p *DirectPublisher) PublishJSON(topic string, data interface{}) error {
    payload, err := json.Marshal(data)
    if err != nil {
        return err
    }
    return p.Publish(topic, payload)
}

func (p *DirectPublisher) Terminate() {
    p.publisher.Terminate(10 * time.Second)
}
```

### Persistent Publisher with Acknowledgments

```go
type PersistentPublisher struct {
    service   solace.MessagingService
    publisher solace.PersistentMessagePublisher
    ackChan   chan PublishReceipt
}

type PublishReceipt struct {
    MessageID string
    Persisted bool
    Error     error
}

func NewPersistentPublisher(service solace.MessagingService) (*PersistentPublisher, error) {
    ackChan := make(chan PublishReceipt, 1000)

    publisher, err := service.CreatePersistentMessagePublisherBuilder().
        OnBackPressureWait(100).
        Build()
    if err != nil {
        return nil, err
    }

    // Set up receipt listener
    publisher.SetMessagePublishReceiptListener(func(receipt solace.PublishReceipt) {
        result := PublishReceipt{
            MessageID: receipt.GetUserContext().(string),
            Persisted: receipt.IsPersisted(),
        }
        if !receipt.IsPersisted() {
            result.Error = receipt.GetError()
        }
        ackChan <- result
    })

    if err := publisher.Start(); err != nil {
        return nil, err
    }

    return &PersistentPublisher{
        service:   service,
        publisher: publisher,
        ackChan:   ackChan,
    }, nil
}

func (p *PersistentPublisher) PublishAsync(queue string, payload []byte) (string, error) {
    messageID := uuid.New().String()

    msg, err := p.service.MessageBuilder().
        WithProperty("message-id", messageID).
        BuildWithByteArrayPayload(payload)
    if err != nil {
        return "", err
    }

    // Set correlation context
    p.publisher.PublishWithContext(
        msg,
        resource.QueueDurableExclusive(queue),
        messageID, // userContext for correlation
    )

    return messageID, nil
}

func (p *PersistentPublisher) WaitForReceipt(ctx context.Context) (*PublishReceipt, error) {
    select {
    case receipt := <-p.ackChan:
        return &receipt, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

## Subscribing

### Direct Subscriber with Goroutines

```go
type DirectSubscriber struct {
    service  solace.MessagingService
    receiver solace.DirectMessageReceiver
    handlers map[string]MessageHandler
    mu       sync.RWMutex
}

type MessageHandler func(msg message.InboundMessage) error

func NewDirectSubscriber(service solace.MessagingService) (*DirectSubscriber, error) {
    return &DirectSubscriber{
        service:  service,
        handlers: make(map[string]MessageHandler),
    }, nil
}

func (s *DirectSubscriber) Subscribe(patterns []string, handler MessageHandler) error {
    var subscriptions []resource.Subscription
    for _, pattern := range patterns {
        subscriptions = append(subscriptions, resource.TopicSubscriptionOf(pattern))
        s.mu.Lock()
        s.handlers[pattern] = handler
        s.mu.Unlock()
    }

    receiver, err := s.service.CreateDirectMessageReceiverBuilder().
        WithSubscriptions(subscriptions...).
        Build()
    if err != nil {
        return err
    }

    // Start receiver
    if err := receiver.Start(); err != nil {
        return err
    }
    s.receiver = receiver

    // Receive messages asynchronously
    go func() {
        for {
            msg, err := receiver.ReceiveMessage(-1) // Block indefinitely
            if err != nil {
                log.Printf("Receive error: %v", err)
                return
            }

            // Handle in goroutine for concurrency
            go s.handleMessage(msg)
        }
    }()

    return nil
}

func (s *DirectSubscriber) handleMessage(msg message.InboundMessage) {
    topic := msg.GetDestinationName()
    
    s.mu.RLock()
    handler, ok := s.handlers[topic]
    s.mu.RUnlock()

    if ok {
        if err := handler(msg); err != nil {
            log.Printf("Handler error for %s: %v", topic, err)
        }
    }
}
```

### Queue Consumer with Worker Pool

```go
type QueueConsumer struct {
    service  solace.MessagingService
    receiver solace.PersistentMessageReceiver
    workers  int
}

func NewQueueConsumer(service solace.MessagingService, queueName string, workers int) (*QueueConsumer, error) {
    receiver, err := service.CreatePersistentMessageReceiverBuilder().
        WithMessageAutoAcknowledgement(). // Or WithMessageClientAcknowledgement()
        Build(resource.QueueDurableExclusive(queueName))
    if err != nil {
        return nil, err
    }

    return &QueueConsumer{
        service:  service,
        receiver: receiver,
        workers:  workers,
    }, nil
}

func (c *QueueConsumer) Start(ctx context.Context, handler MessageHandler) error {
    if err := c.receiver.Start(); err != nil {
        return err
    }

    // Create worker pool
    msgChan := make(chan message.InboundMessage, c.workers*10)

    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < c.workers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for msg := range msgChan {
                if err := handler(msg); err != nil {
                    log.Printf("Worker %d error: %v", workerID, err)
                }
            }
        }(i)
    }

    // Receive loop
    go func() {
        for {
            select {
            case <-ctx.Done():
                close(msgChan)
                return
            default:
                msg, err := c.receiver.ReceiveMessage(1000) // 1 second timeout
                if err != nil {
                    if solace.IsTimeout(err) {
                        continue
                    }
                    log.Printf("Receive error: %v", err)
                    return
                }
                if msg != nil {
                    msgChan <- msg
                }
            }
        }
    }()

    wg.Wait()
    return nil
}
```

## Context and Cancellation

```go
func ProcessWithTimeout(ctx context.Context, service solace.MessagingService) error {
    // Create timeout context
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Connect with context
    if err := service.ConnectWithContext(ctx); err != nil {
        return fmt.Errorf("connect failed: %w", err)
    }

    // Publish with context
    publisher, _ := service.CreateDirectMessagePublisherBuilder().Build()
    publisher.Start()

    select {
    case <-ctx.Done():
        publisher.Terminate(5 * time.Second)
        return ctx.Err()
    default:
        // Continue processing
    }

    return nil
}
```

## Error Handling

```go
func HandleSolaceError(err error) {
    if err == nil {
        return
    }

    // Check for specific error types
    switch {
    case solace.IsTimeout(err):
        log.Println("Operation timed out, retrying...")
        // Implement retry logic

    case solace.IsDisconnected(err):
        log.Println("Disconnected from broker")
        // Wait for reconnection

    case solace.IsCannotSend(err):
        log.Println("Cannot send message, publisher flow blocked")
        // Back off and retry

    default:
        log.Printf("Unexpected error: %v", err)
    }
}

// Retry wrapper
func WithRetry(ctx context.Context, maxRetries int, fn func() error) error {
    var err error
    for i := 0; i < maxRetries; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        if !solace.IsRetryable(err) {
            return err
        }

        // Exponential backoff
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(time.Duration(1<<i) * time.Second):
        }
    }
    return fmt.Errorf("max retries exceeded: %w", err)
}
```

## Kubernetes Patterns

### Liveness and Readiness

```go
type SolaceHealthChecker struct {
    service solace.MessagingService
}

func (h *SolaceHealthChecker) IsAlive() bool {
    return h.service != nil
}

func (h *SolaceHealthChecker) IsReady() bool {
    return h.service != nil && h.service.IsConnected()
}

// HTTP handlers for K8s probes
func (h *SolaceHealthChecker) LivenessHandler(w http.ResponseWriter, r *http.Request) {
    if h.IsAlive() {
        w.WriteHeader(http.StatusOK)
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
    }
}

func (h *SolaceHealthChecker) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
    if h.IsReady() {
        w.WriteHeader(http.StatusOK)
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
    }
}
```

### Graceful Shutdown

```go
func RunWithGracefulShutdown(service solace.MessagingService) {
    ctx, cancel := context.WithCancel(context.Background())

    // Handle signals
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        sig := <-sigChan
        log.Printf("Received signal: %v", sig)
        cancel()
    }()

    // Run application
    if err := runApp(ctx, service); err != nil {
        log.Printf("App error: %v", err)
    }

    // Graceful disconnect
    log.Println("Shutting down gracefully...")
    service.Disconnect()
    log.Println("Disconnected")
}
```

## Reference Links

- **Go API Documentation**: https://docs.solace.com/API/Messaging-APIs/Go-API/go-home.htm
- **API Reference**: https://pkg.go.dev/solace.dev/go/messaging
- **Tutorials**: https://tutorials.solace.dev/go
- **GitHub Samples**: https://github.com/SolaceSamples/solace-samples-go

## Further Reading

- [Concurrency Patterns](./references/concurrency.md)
- [Kubernetes Deployment](./references/kubernetes.md)
- [Error Handling](./references/errors.md)
