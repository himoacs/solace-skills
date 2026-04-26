---
name: solace-app-review
description: 'Solace application code review and best practices audit. Use for reviewing messaging patterns, error handling, reconnection logic, message acknowledgment patterns, and identifying anti-patterns in Solace client applications.'
argument-hint: 'java, python, javascript, or go'
---

# Solace Application Review

Expert guidance for auditing Solace client applications against best practices.

## When to Use

- **Code Review**: Audit messaging code quality
- **Pre-Production**: Verify application readiness
- **Anti-Pattern Detection**: Identify common mistakes
- **Performance Review**: Check efficiency patterns

## Review Checklist

### Connection Management

```markdown
## Connection Review Checklist

### Initialization
- [ ] SolClient factory initialized once at startup
- [ ] Context created with appropriate threading model
- [ ] Session properties configured for environment
- [ ] No hardcoded credentials (use environment/secrets)

### Reconnection
- [ ] Automatic reconnection enabled
- [ ] Reconnect retries set appropriately (-1 for infinite)
- [ ] Reconnect delay configured (3-5 seconds typical)
- [ ] Reapply subscriptions enabled
- [ ] Session event handlers implemented

### Cleanup
- [ ] Resources disposed in reverse order of creation
- [ ] Graceful disconnect on shutdown
- [ ] Shutdown hooks registered for cleanup
```

**Good Example**:
```java
// ✅ Proper connection handling
SessionProperties sessionProps = new SessionProperties();
sessionProps.setProperty(SessionProperties.HOST, config.getHost());
sessionProps.setProperty(SessionProperties.RECONNECT_RETRIES, "-1");
sessionProps.setProperty(SessionProperties.RECONNECT_RETRY_WAIT_MS, "3000");
sessionProps.setProperty(SessionProperties.REAPPLY_SUBSCRIPTIONS, "true");

session = JCSMPFactory.onlyInstance().createSession(sessionProps, null, eventHandler);
session.connect();

// Shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    if (session != null) session.closeSession();
}));
```

**Anti-Pattern**:
```java
// ❌ Missing reconnection handling
SessionProperties sessionProps = new SessionProperties();
sessionProps.setProperty(SessionProperties.HOST, "hardcoded-host:55555");
sessionProps.setProperty(SessionProperties.USERNAME, "admin"); // Hardcoded!
session.connect();
// No shutdown hook, no reconnect config
```

### Publishing

```markdown
## Publishing Review Checklist

### Message Construction
- [ ] Messages reused where possible (high throughput)
- [ ] Proper delivery mode selected (direct vs persistent)
- [ ] Correlation IDs set for tracking
- [ ] TTL configured if messages are time-sensitive

### Error Handling
- [ ] Publication errors handled
- [ ] Acknowledgment callbacks implemented (guaranteed)
- [ ] Back-pressure handled gracefully
- [ ] Would-block conditions managed

### Performance
- [ ] Batching considered for high volume
- [ ] Message size appropriate (not too large)
- [ ] No unnecessary serialization/deserialization
```

**Good Example**:
```java
// ✅ Proper guaranteed publishing with ack handling
producer.setMessagePublishReceipt(receipt -> {
    if (receipt.getStatus() == MessageAckStatus.ACK) {
        log.debug("Message acknowledged: {}", receipt.getCorrelationId());
    } else {
        log.error("Message failed: {}", receipt.getStatus());
        retryService.enqueue(receipt.getMessage());
    }
});

try {
    producer.send(message, topic);
} catch (JCSMPException e) {
    if (e instanceof JCSMPTransportException) {
        // Connection issue - will auto-reconnect
        log.warn("Transport error, message queued for retry");
        retryService.enqueue(message);
    } else {
        throw e;
    }
}
```

**Anti-Pattern**:
```java
// ❌ No error handling, no ack listening
producer.send(message, topic);
// Assumes success - messages can be silently lost!
```

### Subscribing

```markdown
## Subscribing Review Checklist

### Subscription Setup
- [ ] Topic patterns are specific (not overly broad)
- [ ] No subscription to `>` alone
- [ ] Wildcards used appropriately
- [ ] Subscriptions reapplied on reconnect

### Message Handling
- [ ] Messages processed asynchronously (not blocking callback)
- [ ] Exception handling in message callback
- [ ] No heavy processing in callback thread
- [ ] Acknowledgments sent after successful processing

### Queue Consumers
- [ ] Flow control handled
- [ ] Client acknowledgment used (not auto-ack for important messages)
- [ ] Message redelivery considered
- [ ] Durable subscriptions used where needed
```

**Good Example**:
```java
// ✅ Proper async message handling with ack
consumer.setMessageCallback(message -> {
    executor.submit(() -> {
        try {
            processMessage(message);
            message.ack(); // Only ack after successful processing
        } catch (Exception e) {
            log.error("Processing failed, message will be redelivered", e);
            // Don't ack - message will be redelivered
        }
    });
});
```

**Anti-Pattern**:
```java
// ❌ Blocking callback, auto-ack even on failure
consumer.setMessageCallback(message -> {
    Thread.sleep(5000); // Blocks callback thread!
    processMessage(message); // If this fails, message is still acked
});
consumer.setAutoAck(true); // Messages acked before processing completes
```

### Error Handling

```markdown
## Error Handling Review Checklist

### Exception Handling
- [ ] Specific exception types caught and handled
- [ ] Transient vs permanent errors distinguished
- [ ] Retry logic with backoff implemented
- [ ] Circuit breaker for external calls

### Logging
- [ ] Errors logged with context
- [ ] Correlation IDs included in logs
- [ ] Stack traces only for unexpected errors
- [ ] Metrics/alerts for error rates
```

**Good Example**:
```java
// ✅ Proper error handling with categorization
try {
    publisher.send(message, topic);
} catch (JCSMPTransportException e) {
    // Transient - retry with backoff
    log.warn("Transport error, scheduling retry: {}", e.getMessage());
    retryWithBackoff(() -> publisher.send(message, topic));
} catch (JCSMPTimeoutException e) {
    // Timeout - may retry
    log.warn("Timeout publishing, will retry: {}", correlationId);
    retryWithBackoff(() -> publisher.send(message, topic));
} catch (JCSMPException e) {
    // Permanent failure
    log.error("Permanent failure publishing message: {}", correlationId, e);
    deadLetterService.store(message);
}
```

### Thread Safety

```markdown
## Thread Safety Review Checklist

- [ ] Session shared safely or per-thread
- [ ] Message objects not shared between threads
- [ ] Thread-safe collections for shared state
- [ ] No race conditions in callback handling
- [ ] Executor services properly managed
```

## Common Anti-Patterns

### 1. Fire and Forget

```java
// ❌ BAD: No error handling, no ack
producer.send(message, topic);
// Message may be lost!
```

### 2. Blocking Callbacks

```java
// ❌ BAD: Blocking the callback thread
messageConsumer.setMessageCallback(msg -> {
    callExternalService(msg); // May take seconds!
    databaseInsert(msg);      // More blocking!
});
```

### 3. Overly Broad Subscriptions

```java
// ❌ BAD: Subscribes to everything
session.addSubscription(JCSMPFactory.onlyInstance()
    .createTopic(">")); // Receives ALL messages!
```

### 4. Ignoring Session Events

```java
// ❌ BAD: No event handler
session = factory.createSession(props, null, null); // No event handler!
```

### 5. Resource Leaks

```java
// ❌ BAD: Message not released
for (int i = 0; i < 10000; i++) {
    XMLMessage msg = producer.createMessage(); // Never released!
    producer.send(msg, topic);
}
```

## Review Report Template

```markdown
# Solace Application Review Report

**Application**: [name]
**Date**: [date]
**Reviewer**: [name]

## Summary
- Overall Score: [X/100]
- Critical Issues: [count]
- Warnings: [count]
- Recommendations: [count]

## Findings

### Critical Issues
1. [Issue description]
   - Location: [file:line]
   - Impact: [description]
   - Fix: [recommendation]

### Warnings
1. [Warning description]
   - Location: [file:line]
   - Recommendation: [description]

### Best Practices Not Followed
1. [Practice]
   - Current: [description]
   - Recommended: [description]

## Recommendations
1. [Priority 1 recommendations]
2. [Priority 2 recommendations]

## Conclusion
[Summary and next steps]
```

## Automated Checks

### Static Analysis Rules

```yaml
# Example custom rules for linting
rules:
  - id: solace-missing-reconnect
    pattern: |
      SessionProperties props = new SessionProperties();
      # Missing RECONNECT_RETRIES
    severity: WARNING
    message: "Missing reconnection configuration"

  - id: solace-blocking-callback
    pattern: |
      setMessageCallback($MSG -> {
        ...
        Thread.sleep($N);
        ...
      })
    severity: ERROR
    message: "Blocking operation in message callback"

  - id: solace-hardcoded-credentials
    pattern: |
      setProperty(SessionProperties.PASSWORD, "$LITERAL")
    severity: CRITICAL
    message: "Hardcoded password detected"
```

## Reference Links

- **Best Practices**: https://docs.solace.com/Best-Practices/Best-Practices.htm
- **API Documentation**: https://docs.solace.com/API/API-Developer-Guide.htm
- **Error Handling**: https://docs.solace.com/API/Error-Handling.htm

## Further Reading

- [Performance Checklist](./references/performance-checklist.md)
- [Security Review](./references/security-review.md)
- [Production Readiness](./references/production-checklist.md)
