---
name: solace-python-development
description: 'Solace Python API development guidance. Use for solace-pubsubplus native API, direct and persistent messaging, asyncio async patterns, request-reply, data processing pipelines, and integration with pandas/numpy.'
argument-hint: 'publish, subscribe, async, request-reply, or pipeline'
---

# Solace Python Development

Expert guidance for building Python applications with Solace PubSub+ messaging.

## When to Use

- **Messaging Applications**: Pub/sub with Python
- **Data Pipelines**: Processing event streams
- **Async Patterns**: asyncio integration
- **Request-Reply**: Synchronous patterns over messaging
- **Integration**: pandas, numpy, data science workflows

## Installation

```bash
pip install solace-pubsubplus
```

## Basic Connection

```python
from solace.messaging.messaging_service import MessagingService
from solace.messaging.config.solace_properties import service_properties

# Connection properties
broker_props = {
    service_properties.TRANSPORT_LAYER_HOST: "tcp://localhost:55555",
    service_properties.SERVICE_VPN_NAME: "default",
    service_properties.AUTHENTICATION_BASIC_USER_NAME: "user",
    service_properties.AUTHENTICATION_BASIC_PASSWORD: "password",
}

# Build and connect
messaging_service = MessagingService.builder() \
    .from_properties(broker_props) \
    .with_reconnection_retry_strategy(
        RetryStrategy.parametrized_retry(20, 3000)  # 20 retries, 3sec interval
    ) \
    .build()

messaging_service.connect()
print(f"Connected: {messaging_service.is_connected}")
```

## Direct Messaging (Non-Persistent)

### Publisher

```python
from solace.messaging.publisher.direct_message_publisher import DirectMessagePublisher
from solace.messaging.resources.topic import Topic

def create_direct_publisher(service: MessagingService) -> DirectMessagePublisher:
    publisher = service.create_direct_message_publisher_builder() \
        .on_back_pressure_wait(max_messages=100) \
        .build()
    publisher.start()
    return publisher

def publish_message(publisher: DirectMessagePublisher, 
                    service: MessagingService,
                    topic_name: str, 
                    payload: str):
    topic = Topic.of(topic_name)
    message = service.message_builder() \
        .with_application_message_id(str(uuid.uuid4())) \
        .build(payload)
    
    publisher.publish(message, topic)
    print(f"Published to {topic_name}")
```

### Subscriber

```python
from solace.messaging.receiver.direct_message_receiver import DirectMessageReceiver
from solace.messaging.receiver.message_receiver import InboundMessage, MessageHandler
from solace.messaging.resources.topic_subscription import TopicSubscription

class MyMessageHandler(MessageHandler):
    def on_message(self, message: InboundMessage):
        topic = message.get_destination_name()
        payload = message.get_payload_as_string()
        print(f"Received from {topic}: {payload}")

def create_direct_subscriber(service: MessagingService, 
                             topic_patterns: list[str]) -> DirectMessageReceiver:
    subscriptions = [TopicSubscription.of(pattern) for pattern in topic_patterns]
    
    receiver = service.create_direct_message_receiver_builder() \
        .with_subscriptions(subscriptions) \
        .build()
    
    receiver.start()
    receiver.receive_async(MyMessageHandler())
    
    return receiver
```

## Persistent Messaging (Guaranteed)

### Publisher with Acknowledgments

```python
from solace.messaging.publisher.persistent_message_publisher import PersistentMessagePublisher
from solace.messaging.publisher.publish_receipt import PublishReceipt

class PublishReceiptListener:
    def on_publish_receipt(self, publish_receipt: PublishReceipt):
        if publish_receipt.is_persisted:
            print(f"Message persisted: {publish_receipt.timestamp}")
        else:
            print(f"Message failed: {publish_receipt.exception}")

def create_persistent_publisher(service: MessagingService) -> PersistentMessagePublisher:
    publisher = service.create_persistent_message_publisher_builder() \
        .build()
    
    publisher.set_publish_receipt_listener(PublishReceiptListener())
    publisher.start()
    
    return publisher

def publish_persistent(publisher: PersistentMessagePublisher,
                       service: MessagingService,
                       queue_name: str,
                       payload: str):
    from solace.messaging.resources.queue import Queue
    
    queue = Queue.durable_exclusive_queue(queue_name)
    message = service.message_builder() \
        .with_application_message_id(str(uuid.uuid4())) \
        .build(payload)
    
    publisher.publish(message, queue)
```

### Subscriber with Manual Acknowledgment

```python
from solace.messaging.receiver.persistent_message_receiver import PersistentMessageReceiver
from solace.messaging.resources.queue import Queue

def create_persistent_subscriber(service: MessagingService,
                                  queue_name: str) -> PersistentMessageReceiver:
    queue = Queue.durable_exclusive_queue(queue_name)
    
    receiver = service.create_persistent_message_receiver_builder() \
        .with_message_auto_acknowledgment() \  # Or manual
        .build(queue)
    
    receiver.start()
    return receiver

def receive_with_manual_ack(receiver: PersistentMessageReceiver):
    while True:
        message = receiver.receive_message(timeout=5000)  # 5 seconds
        if message:
            try:
                payload = message.get_payload_as_string()
                process_message(payload)
                receiver.ack(message)  # Manual acknowledgment
            except Exception as e:
                print(f"Processing failed: {e}")
                # Message will be redelivered if not acked
```

## Async Patterns with asyncio

### Async Publisher

```python
import asyncio
from solace.messaging.messaging_service import MessagingService

async def async_publish(service: MessagingService, 
                        topic: str, 
                        messages: list[str]):
    publisher = service.create_direct_message_publisher_builder() \
        .on_back_pressure_wait(100) \
        .build()
    publisher.start()
    
    topic_obj = Topic.of(topic)
    
    for payload in messages:
        message = service.message_builder().build(payload)
        publisher.publish(message, topic_obj)
        await asyncio.sleep(0)  # Yield to event loop
    
    publisher.terminate()

# Usage
async def main():
    service = create_service()
    service.connect()
    
    messages = [f"Message {i}" for i in range(100)]
    await async_publish(service, "async/test", messages)

asyncio.run(main())
```

### Async Consumer with Queue

```python
import asyncio
from queue import Queue as ThreadQueue
from threading import Thread

class AsyncMessageHandler(MessageHandler):
    def __init__(self, async_queue: asyncio.Queue):
        self.async_queue = async_queue
        self.loop = asyncio.get_event_loop()
    
    def on_message(self, message: InboundMessage):
        # Thread-safe put to async queue
        asyncio.run_coroutine_threadsafe(
            self.async_queue.put(message),
            self.loop
        )

async def async_consumer(service: MessagingService, topic_pattern: str):
    message_queue = asyncio.Queue()
    handler = AsyncMessageHandler(message_queue)
    
    receiver = service.create_direct_message_receiver_builder() \
        .with_subscriptions([TopicSubscription.of(topic_pattern)]) \
        .build()
    receiver.start()
    receiver.receive_async(handler)
    
    while True:
        message = await message_queue.get()
        payload = message.get_payload_as_string()
        await process_async(payload)
```

## Request-Reply Pattern

### Requestor

```python
from solace.messaging.publisher.request_reply.request_reply_message_publisher import RequestReplyMessagePublisher

def send_request(service: MessagingService, 
                 request_topic: str, 
                 request_payload: str,
                 timeout_ms: int = 5000) -> str:
    
    publisher = service.create_request_reply_message_publisher_builder().build()
    publisher.start()
    
    topic = Topic.of(request_topic)
    request = service.message_builder().build(request_payload)
    
    # Blocking request
    reply = publisher.publish_await_response(
        request_message=request,
        request_destination=topic,
        reply_timeout=timeout_ms
    )
    
    if reply:
        return reply.get_payload_as_string()
    return None
```

### Replier

```python
from solace.messaging.receiver.request_reply.request_reply_message_receiver import RequestReplyMessageReceiver

def create_replier(service: MessagingService, request_topic: str):
    receiver = service.create_request_reply_message_receiver_builder() \
        .with_subscriptions([TopicSubscription.of(request_topic)]) \
        .build()
    receiver.start()
    
    return receiver

def handle_requests(receiver: RequestReplyMessageReceiver, service: MessagingService):
    while True:
        request = receiver.receive_message(timeout=10000)
        if request:
            # Process request
            request_payload = request.get_payload_as_string()
            response_payload = process_request(request_payload)
            
            # Build and send reply
            reply = service.message_builder().build(response_payload)
            receiver.reply(request, reply)
```

## Data Pipeline Integration

### pandas Integration

```python
import pandas as pd
import json
from datetime import datetime

class DataPipelinePublisher:
    def __init__(self, service: MessagingService, topic_prefix: str):
        self.service = service
        self.topic_prefix = topic_prefix
        self.publisher = service.create_direct_message_publisher_builder() \
            .on_back_pressure_wait(1000) \
            .build()
        self.publisher.start()
    
    def publish_dataframe(self, df: pd.DataFrame, topic_suffix: str):
        """Publish DataFrame rows as individual messages"""
        topic = Topic.of(f"{self.topic_prefix}/{topic_suffix}")
        
        for _, row in df.iterrows():
            payload = row.to_json()
            message = self.service.message_builder() \
                .with_property("timestamp", datetime.utcnow().isoformat()) \
                .build(payload)
            self.publisher.publish(message, topic)
    
    def publish_batch(self, df: pd.DataFrame, topic_suffix: str, batch_size: int = 100):
        """Publish DataFrame in batches"""
        topic = Topic.of(f"{self.topic_prefix}/{topic_suffix}")
        
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            payload = batch.to_json(orient='records')
            message = self.service.message_builder() \
                .with_property("batch_start", str(i)) \
                .with_property("batch_size", str(len(batch))) \
                .build(payload)
            self.publisher.publish(message, topic)


class DataPipelineConsumer:
    def __init__(self, service: MessagingService):
        self.service = service
        self.data_buffer = []
    
    def consume_to_dataframe(self, 
                             queue_name: str, 
                             max_messages: int = 1000,
                             timeout_ms: int = 5000) -> pd.DataFrame:
        """Consume messages into a DataFrame"""
        queue = Queue.durable_exclusive_queue(queue_name)
        receiver = self.service.create_persistent_message_receiver_builder() \
            .with_message_auto_acknowledgment() \
            .build(queue)
        receiver.start()
        
        records = []
        for _ in range(max_messages):
            message = receiver.receive_message(timeout=timeout_ms)
            if not message:
                break
            
            payload = json.loads(message.get_payload_as_string())
            payload['_received_at'] = datetime.utcnow()
            payload['_topic'] = message.get_destination_name()
            records.append(payload)
        
        receiver.terminate()
        return pd.DataFrame(records)
```

## Error Handling

```python
from solace.messaging.errors.pubsubplus_client_error import PubSubPlusClientError

class RobustMessageHandler(MessageHandler):
    def __init__(self, service: MessagingService, dlq_name: str):
        self.service = service
        self.dlq = Queue.durable_non_exclusive_queue(dlq_name)
        self.dlq_publisher = service.create_persistent_message_publisher_builder().build()
        self.dlq_publisher.start()
    
    def on_message(self, message: InboundMessage):
        try:
            payload = message.get_payload_as_string()
            self.process(payload)
        except RecoverableError as e:
            # Will be redelivered
            raise e
        except Exception as e:
            # Send to DLQ
            self.send_to_dlq(message, str(e))
    
    def send_to_dlq(self, original: InboundMessage, error: str):
        dlq_message = self.service.message_builder() \
            .with_property("original_topic", original.get_destination_name()) \
            .with_property("error", error) \
            .with_property("failed_at", datetime.utcnow().isoformat()) \
            .build(original.get_payload_as_bytes())
        
        self.dlq_publisher.publish(dlq_message, self.dlq)


# Connection error handling
def connect_with_retry(properties: dict, max_retries: int = 5) -> MessagingService:
    for attempt in range(max_retries):
        try:
            service = MessagingService.builder() \
                .from_properties(properties) \
                .build()
            service.connect()
            return service
        except PubSubPlusClientError as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

## Best Practices

### Context Manager Pattern

```python
from contextlib import contextmanager

@contextmanager
def solace_session(properties: dict):
    service = MessagingService.builder() \
        .from_properties(properties) \
        .build()
    service.connect()
    try:
        yield service
    finally:
        service.disconnect()

# Usage
with solace_session(broker_props) as service:
    publish_message(service, "test/topic", "Hello")
```

### Logging Configuration

```python
import logging

# Enable Solace API logging
logging.basicConfig(level=logging.INFO)
solace_logger = logging.getLogger('solace.messaging')
solace_logger.setLevel(logging.DEBUG)
```

## Reference Links

- **Python API Documentation**: https://docs.solace.com/API/Messaging-APIs/Python-API/python-home.htm
- **API Reference**: https://docs.solace.com/API/python-api/autoapi/solace/index.html
- **Tutorials**: https://tutorials.solace.dev/python
- **GitHub Samples**: https://github.com/SolaceSamples/solace-samples-python

## Further Reading

- [Async Patterns](./references/async-patterns.md)
- [Data Pipeline Examples](./references/data-pipelines.md)
- [Error Handling](./references/error-handling.md)
