---
name: solace-c-development
description: 'Solace C API development guidance. Use for SolClient C native API, embedded systems integration, low-latency messaging patterns, memory management, callback patterns, and cross-compilation for ARM/x86.'
argument-hint: 'embedded, callbacks, memory, or cross-compile'
---

# Solace C Development

Expert guidance for building C applications with Solace PubSub+ native C API.

## When to Use

- **Embedded Systems**: Resource-constrained devices
- **Low Latency**: Performance-critical applications
- **System Integration**: C/C++ native applications
- **Cross-Platform**: ARM, x86, various OS

## Installation

Download the C API from the Solace downloads portal for your target platform.

```bash
# Extract and set environment
export SOLCLIENT_DIR=/path/to/solclient-7.x.x
export LD_LIBRARY_PATH=$SOLCLIENT_DIR/lib:$LD_LIBRARY_PATH
```

### CMake Integration

```cmake
cmake_minimum_required(VERSION 3.10)
project(solace_app C)

set(CMAKE_C_STANDARD 11)

# Find Solace C API
set(SOLCLIENT_DIR $ENV{SOLCLIENT_DIR})
include_directories(${SOLCLIENT_DIR}/include)
link_directories(${SOLCLIENT_DIR}/lib)

add_executable(publisher src/publisher.c)
target_link_libraries(publisher solclient pthread)

add_executable(subscriber src/subscriber.c)
target_link_libraries(subscriber solclient pthread)
```

## Initialization

```c
#include "solclient/solClient.h"
#include "solclient/solClientMsg.h"

/* Initialize the Solace API */
int initialize_solace(void) {
    solClient_returnCode_t rc;
    
    /* Initialize the API */
    solClient_opaqueContext_pt context_p;
    solClient_context_createFuncInfo_t contextFuncInfo = 
        SOLCLIENT_CONTEXT_CREATEFUNC_INITIALIZER;
    
    rc = solClient_initialize(SOLCLIENT_LOG_DEFAULT_FILTER, NULL);
    if (rc != SOLCLIENT_OK) {
        printf("solClient_initialize failed: %d\n", rc);
        return -1;
    }
    
    /* Create context */
    rc = solClient_context_create(SOLCLIENT_CONTEXT_PROPS_DEFAULT_WITH_CREATE_THREAD,
                                   &context_p, &contextFuncInfo, sizeof(contextFuncInfo));
    if (rc != SOLCLIENT_OK) {
        printf("solClient_context_create failed: %d\n", rc);
        return -1;
    }
    
    return 0;
}
```

## Connection

### Session Properties

```c
typedef struct {
    solClient_opaqueContext_pt context;
    solClient_opaqueSession_pt session;
} solace_connection_t;

int create_session(solace_connection_t *conn, const char *host, 
                   const char *vpn, const char *username, const char *password) {
    solClient_returnCode_t rc;
    
    /* Session properties */
    const char *sessionProps[] = {
        SOLCLIENT_SESSION_PROP_HOST, host,
        SOLCLIENT_SESSION_PROP_VPN_NAME, vpn,
        SOLCLIENT_SESSION_PROP_USERNAME, username,
        SOLCLIENT_SESSION_PROP_PASSWORD, password,
        SOLCLIENT_SESSION_PROP_CONNECT_TIMEOUT_MS, "10000",
        SOLCLIENT_SESSION_PROP_RECONNECT_RETRIES, "-1",  /* Infinite */
        SOLCLIENT_SESSION_PROP_RECONNECT_RETRY_WAIT_MS, "3000",
        SOLCLIENT_SESSION_PROP_REAPPLY_SUBSCRIPTIONS, "true",
        NULL
    };
    
    /* Session function info */
    solClient_session_createFuncInfo_t sessionFuncInfo = 
        SOLCLIENT_SESSION_CREATEFUNC_INITIALIZER;
    sessionFuncInfo.rxMsgInfo.callback_p = message_receive_callback;
    sessionFuncInfo.rxMsgInfo.user_p = conn;
    sessionFuncInfo.eventInfo.callback_p = session_event_callback;
    sessionFuncInfo.eventInfo.user_p = conn;
    
    /* Create session */
    rc = solClient_session_create(sessionProps, conn->context, 
                                   &conn->session, &sessionFuncInfo,
                                   sizeof(sessionFuncInfo));
    if (rc != SOLCLIENT_OK) {
        printf("solClient_session_create failed: %d\n", rc);
        return -1;
    }
    
    /* Connect */
    rc = solClient_session_connect(conn->session);
    if (rc != SOLCLIENT_OK) {
        printf("solClient_session_connect failed: %d\n", rc);
        return -1;
    }
    
    return 0;
}
```

### Event Callback

```c
solClient_rxMsgCallback_returnCode_t session_event_callback(
    solClient_opaqueSession_pt session_p,
    solClient_session_eventCallbackInfo_pt eventInfo_p,
    void *user_p) {
    
    switch (eventInfo_p->sessionEvent) {
        case SOLCLIENT_SESSION_EVENT_UP_NOTICE:
            printf("Session connected\n");
            break;
            
        case SOLCLIENT_SESSION_EVENT_CONNECT_FAILED_ERROR:
            printf("Connection failed: %s\n", 
                   solClient_session_eventToString(eventInfo_p->sessionEvent));
            break;
            
        case SOLCLIENT_SESSION_EVENT_RECONNECTING_NOTICE:
            printf("Reconnecting...\n");
            break;
            
        case SOLCLIENT_SESSION_EVENT_RECONNECTED_NOTICE:
            printf("Reconnected\n");
            break;
            
        case SOLCLIENT_SESSION_EVENT_DOWN_ERROR:
            printf("Session down: %s\n",
                   eventInfo_p->info_p ? eventInfo_p->info_p : "");
            break;
            
        default:
            printf("Session event: %d\n", eventInfo_p->sessionEvent);
            break;
    }
    
    return SOLCLIENT_CALLBACK_OK;
}
```

## Publishing

### Direct Messages

```c
int publish_direct(solClient_opaqueSession_pt session, 
                   const char *topic, const void *data, size_t len) {
    solClient_returnCode_t rc;
    solClient_opaqueMsg_pt msg = NULL;
    solClient_destination_t destination;
    
    /* Allocate message */
    rc = solClient_msg_alloc(&msg);
    if (rc != SOLCLIENT_OK) {
        return -1;
    }
    
    /* Set delivery mode */
    rc = solClient_msg_setDeliveryMode(msg, SOLCLIENT_DELIVERY_MODE_DIRECT);
    
    /* Set destination */
    destination.destType = SOLCLIENT_TOPIC_DESTINATION;
    destination.dest = topic;
    rc = solClient_msg_setDestination(msg, &destination, sizeof(destination));
    
    /* Set payload */
    rc = solClient_msg_setBinaryAttachment(msg, data, len);
    
    /* Send */
    rc = solClient_session_sendMsg(session, msg);
    
    /* Free message */
    solClient_msg_free(&msg);
    
    return (rc == SOLCLIENT_OK) ? 0 : -1;
}

/* Usage */
const char *json = "{\"orderId\": \"12345\"}";
publish_direct(session, "acme/orders/created/v1", json, strlen(json));
```

### Guaranteed Messages with Acknowledgment

```c
typedef struct {
    void *correlation_key;
    int ack_received;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
} ack_context_t;

void message_ack_callback(solClient_opaqueSession_pt session_p,
                          solClient_msgAck_pt msgAck_p,
                          void *user_p) {
    ack_context_t *ctx = (ack_context_t *)user_p;
    
    pthread_mutex_lock(&ctx->mutex);
    ctx->ack_received = 1;
    pthread_cond_signal(&ctx->cond);
    pthread_mutex_unlock(&ctx->mutex);
}

int publish_guaranteed(solClient_opaqueSession_pt session,
                       const char *queue_name, const void *data, size_t len,
                       int timeout_ms) {
    solClient_returnCode_t rc;
    solClient_opaqueMsg_pt msg = NULL;
    solClient_destination_t destination;
    ack_context_t ack_ctx = {0};
    
    pthread_mutex_init(&ack_ctx.mutex, NULL);
    pthread_cond_init(&ack_ctx.cond, NULL);
    
    /* Allocate message */
    solClient_msg_alloc(&msg);
    
    /* Set persistent delivery */
    solClient_msg_setDeliveryMode(msg, SOLCLIENT_DELIVERY_MODE_PERSISTENT);
    
    /* Set queue destination */
    destination.destType = SOLCLIENT_QUEUE_DESTINATION;
    destination.dest = queue_name;
    solClient_msg_setDestination(msg, &destination, sizeof(destination));
    
    /* Set payload */
    solClient_msg_setBinaryAttachment(msg, data, len);
    
    /* Set correlation tag for ack tracking */
    solClient_msg_setCorrelationTagPtr(msg, &ack_ctx, sizeof(ack_ctx));
    
    /* Send */
    rc = solClient_session_sendMsg(session, msg);
    
    if (rc == SOLCLIENT_OK) {
        /* Wait for acknowledgment */
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);
        ts.tv_sec += timeout_ms / 1000;
        ts.tv_nsec += (timeout_ms % 1000) * 1000000;
        
        pthread_mutex_lock(&ack_ctx.mutex);
        while (!ack_ctx.ack_received) {
            int wait_rc = pthread_cond_timedwait(&ack_ctx.cond, &ack_ctx.mutex, &ts);
            if (wait_rc == ETIMEDOUT) {
                pthread_mutex_unlock(&ack_ctx.mutex);
                solClient_msg_free(&msg);
                return -1;  /* Timeout */
            }
        }
        pthread_mutex_unlock(&ack_ctx.mutex);
    }
    
    solClient_msg_free(&msg);
    pthread_mutex_destroy(&ack_ctx.mutex);
    pthread_cond_destroy(&ack_ctx.cond);
    
    return (rc == SOLCLIENT_OK) ? 0 : -1;
}
```

## Subscribing

### Message Callback

```c
solClient_rxMsgCallback_returnCode_t message_receive_callback(
    solClient_opaqueSession_pt session_p,
    solClient_opaqueMsg_pt msg_p,
    void *user_p) {
    
    solClient_destination_t destination;
    void *data_p;
    solClient_uint32_t dataSize;
    
    /* Get destination */
    solClient_msg_getDestination(msg_p, &destination, sizeof(destination));
    printf("Received on topic: %s\n", destination.dest);
    
    /* Get payload */
    if (solClient_msg_getBinaryAttachmentPtr(msg_p, &data_p, &dataSize) == SOLCLIENT_OK) {
        printf("Payload (%u bytes): %.*s\n", dataSize, dataSize, (char *)data_p);
    }
    
    /* Get correlation ID if present */
    const char *correlationId;
    if (solClient_msg_getCorrelationId(msg_p, &correlationId) == SOLCLIENT_OK) {
        printf("Correlation ID: %s\n", correlationId);
    }
    
    return SOLCLIENT_CALLBACK_OK;
}

int subscribe_topic(solClient_opaqueSession_pt session, const char *topic) {
    solClient_returnCode_t rc;
    
    rc = solClient_session_topicSubscribeExt(
        session,
        SOLCLIENT_SUBSCRIBE_FLAGS_WAITFORCONFIRM,
        topic);
    
    return (rc == SOLCLIENT_OK) ? 0 : -1;
}
```

### Queue Consumer (Flow)

```c
typedef struct {
    solClient_opaqueFlow_pt flow;
    message_handler_fn handler;
    void *user_data;
} queue_consumer_t;

solClient_rxMsgCallback_returnCode_t flow_message_callback(
    solClient_opaqueFlow_pt flow_p,
    solClient_opaqueMsg_pt msg_p,
    void *user_p) {
    
    queue_consumer_t *consumer = (queue_consumer_t *)user_p;
    void *data_p;
    solClient_uint32_t dataSize;
    
    /* Get payload */
    solClient_msg_getBinaryAttachmentPtr(msg_p, &data_p, &dataSize);
    
    /* Process message */
    int result = consumer->handler(data_p, dataSize, consumer->user_data);
    
    if (result == 0) {
        /* Acknowledge message */
        solClient_flow_sendAck(flow_p, solClient_msg_getMsgId(msg_p));
    }
    /* Else: message will be redelivered */
    
    return SOLCLIENT_CALLBACK_OK;
}

int create_queue_consumer(solClient_opaqueSession_pt session,
                          const char *queue_name,
                          queue_consumer_t *consumer) {
    solClient_returnCode_t rc;
    
    /* Flow properties */
    const char *flowProps[] = {
        SOLCLIENT_FLOW_PROP_BIND_BLOCKING, SOLCLIENT_PROP_ENABLE_VAL,
        SOLCLIENT_FLOW_PROP_BIND_ENTITY_ID, SOLCLIENT_FLOW_PROP_BIND_ENTITY_QUEUE,
        SOLCLIENT_FLOW_PROP_BIND_NAME, queue_name,
        SOLCLIENT_FLOW_PROP_ACKMODE, SOLCLIENT_FLOW_PROP_ACKMODE_CLIENT,
        NULL
    };
    
    /* Flow function info */
    solClient_flow_createFuncInfo_t flowFuncInfo = 
        SOLCLIENT_FLOW_CREATEFUNC_INITIALIZER;
    flowFuncInfo.rxMsgInfo.callback_p = flow_message_callback;
    flowFuncInfo.rxMsgInfo.user_p = consumer;
    
    /* Create flow */
    rc = solClient_session_createFlow(flowProps, session, 
                                       &consumer->flow, &flowFuncInfo,
                                       sizeof(flowFuncInfo));
    
    return (rc == SOLCLIENT_OK) ? 0 : -1;
}
```

## Memory Management

### Buffer Pool Pattern

```c
#define BUFFER_POOL_SIZE 100
#define BUFFER_SIZE 4096

typedef struct {
    char buffers[BUFFER_POOL_SIZE][BUFFER_SIZE];
    int in_use[BUFFER_POOL_SIZE];
    pthread_mutex_t mutex;
} buffer_pool_t;

buffer_pool_t *buffer_pool_create(void) {
    buffer_pool_t *pool = calloc(1, sizeof(buffer_pool_t));
    pthread_mutex_init(&pool->mutex, NULL);
    return pool;
}

char *buffer_pool_acquire(buffer_pool_t *pool) {
    pthread_mutex_lock(&pool->mutex);
    for (int i = 0; i < BUFFER_POOL_SIZE; i++) {
        if (!pool->in_use[i]) {
            pool->in_use[i] = 1;
            pthread_mutex_unlock(&pool->mutex);
            return pool->buffers[i];
        }
    }
    pthread_mutex_unlock(&pool->mutex);
    return NULL;  /* Pool exhausted */
}

void buffer_pool_release(buffer_pool_t *pool, char *buffer) {
    pthread_mutex_lock(&pool->mutex);
    int idx = (buffer - pool->buffers[0]) / BUFFER_SIZE;
    if (idx >= 0 && idx < BUFFER_POOL_SIZE) {
        pool->in_use[idx] = 0;
    }
    pthread_mutex_unlock(&pool->mutex);
}
```

### Message Reuse

```c
/* Pre-allocate message for high-throughput publishing */
solClient_opaqueMsg_pt reusable_msg = NULL;

int init_reusable_message(void) {
    return solClient_msg_alloc(&reusable_msg);
}

int publish_fast(solClient_opaqueSession_pt session,
                 const char *topic, const void *data, size_t len) {
    solClient_destination_t dest;
    
    /* Reset message */
    solClient_msg_reset(reusable_msg);
    
    /* Set properties */
    solClient_msg_setDeliveryMode(reusable_msg, SOLCLIENT_DELIVERY_MODE_DIRECT);
    dest.destType = SOLCLIENT_TOPIC_DESTINATION;
    dest.dest = topic;
    solClient_msg_setDestination(reusable_msg, &dest, sizeof(dest));
    solClient_msg_setBinaryAttachment(reusable_msg, data, len);
    
    return solClient_session_sendMsg(session, reusable_msg);
}
```

## Cross-Compilation

### ARM Target

```bash
# Set cross-compiler
export CC=arm-linux-gnueabihf-gcc
export CXX=arm-linux-gnueabihf-g++

# CMake toolchain file
cat > arm-toolchain.cmake << 'EOF'
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
EOF

# Build
cmake -DCMAKE_TOOLCHAIN_FILE=arm-toolchain.cmake ..
make
```

## Official C API Best Practices (from Solace Documentation)

### Threading Model Selection

```c
/* RECOMMENDATION: Use API-provided context thread whenever possible */

/* Option 1: API-provided context thread (RECOMMENDED) */
/* Context thread blocks in processEvents and handles all callbacks */
solClient_context_createFuncInfo_t contextFuncInfo = 
    SOLCLIENT_CONTEXT_CREATEFUNC_INITIALIZER;
    
const char *contextProps[] = {
    SOLCLIENT_CONTEXT_PROP_CREATE_THREAD, SOLCLIENT_PROP_ENABLE_VAL,
    NULL
};

solClient_context_create(contextProps, &context, &contextFuncInfo, sizeof(contextFuncInfo));

/* Option 2: Application-provided thread (for custom event loops) */
/* Application must call solClient_context_processEvents() and solClient_context_timerTick() */
const char *contextPropsNoThread[] = {
    SOLCLIENT_CONTEXT_PROP_CREATE_THREAD, SOLCLIENT_PROP_DISABLE_VAL,
    NULL
};

/* Threading Models:
 * - One Session, One Context: Simplest, easiest debugging (RECOMMENDED)
 * - Multiple Sessions, One Context: Higher throughput but more processing stress
 * - Multiple Sessions, Multiple Contexts: Best parallel performance but costly context switching
 */

/* Thread Affinity: Pin context thread to CPU for performance */
const char *contextPropsAffinity[] = {
    SOLCLIENT_CONTEXT_PROP_CREATE_THREAD, SOLCLIENT_PROP_ENABLE_VAL,
    SOLCLIENT_CONTEXT_PROP_THREAD_AFFINITY_CPU_LIST, "0,1,2,4,8-10",
    NULL
};
```

### Session Establishment Best Practices

```c
/* HA Failover: Configure reconnect duration for at least 5 minutes */
const char *sessionPropsHA[] = {
    // ... other props ...
    SOLCLIENT_SESSION_PROP_CONNECT_RETRIES, "1",
    SOLCLIENT_SESSION_PROP_RECONNECT_RETRIES, "20",
    SOLCLIENT_SESSION_PROP_RECONNECT_RETRY_WAIT_MS, "3000",
    SOLCLIENT_SESSION_PROP_CONNECT_RETRIES_PER_HOST, "5",
    NULL
};

/* Replication Failover: Use infinite retries (non-deterministic duration) */
const char *sessionPropsReplication[] = {
    SOLCLIENT_SESSION_PROP_RECONNECT_RETRIES, "-1",  /* Infinite */
    NULL
};

/* Host lists for HA (software event broker) */
const char *sessionPropsHostList[] = {
    SOLCLIENT_SESSION_PROP_HOST, "tcp://primary:55555,tcp://backup:55555",
    SOLCLIENT_SESSION_PROP_CONNECT_RETRIES_PER_HOST, "1",  /* Quick failover */
    NULL
};

/* Disable blocking connect when multiple sessions in context */
/* Blocking connect serializes every session connect */
const char *sessionPropsNoBlock[] = {
    SOLCLIENT_SESSION_PROP_CONNECT_BLOCKING, SOLCLIENT_PROP_DISABLE_VAL,
    NULL
};

/* Reapply subscriptions on reconnect (RECOMMENDED) */
const char *sessionPropsReapply[] = {
    SOLCLIENT_SESSION_PROP_REAPPLY_SUBSCRIPTIONS, SOLCLIENT_PROP_ENABLE_VAL,
    NULL
};
```

### Keep-Alive Configuration

```c
/* Keep-alive should be same order of magnitude as broker TCP keep-alive
 * Default broker: 3s idle + 5 probes at 1s = 8s to detect failure
 * Default API: 3s interval, 3 missed = 9s
 * 
 * WARNING: Too aggressive keep-alive causes premature disconnection
 */
const char *sessionPropsKeepalive[] = {
    SOLCLIENT_SESSION_PROP_KEEP_ALIVE_INT_MS, "3000",    /* 3 seconds */
    SOLCLIENT_SESSION_PROP_KEEP_ALIVE_LIMIT, "3",        /* 3 missed */
    NULL
};
```

### Memory Management Best Practices

```c
/* Use static initializer macros for callback structures */
solClient_session_createFuncInfo_t sessionFuncInfo = 
    SOLCLIENT_SESSION_CREATEFUNC_INITIALIZER;
sessionFuncInfo.rxMsgInfo.callback_p = rxMsgCallback;
sessionFuncInfo.rxMsgInfo.user_p = userData;
sessionFuncInfo.eventInfo.callback_p = eventCallback;
sessionFuncInfo.eventInfo.user_p = userData;
/* Using initializer ensures compatibility with future API changes */

/* Reduce memory copies when publishing */
/* Use pointer variants instead of copy variants */
solClient_msg_setBinaryAttachmentPtr(msg, data, len);  /* No copy */
solClient_msg_setTopicPtr(msg, topic);                 /* No copy */
/* vs */
solClient_msg_setBinaryAttachment(msg, data, len);     /* Copies data */

/* Configure TCP buffers for large messages or WAN */
const char *sessionPropsTcp[] = {
    SOLCLIENT_SESSION_PROP_SOCKET_RCV_BUF_SIZE, "150000",  /* 150KB */
    SOLCLIENT_SESSION_PROP_SOCKET_SEND_BUF_SIZE, "90000",  /* 90KB */
    /* Windows: Receive should be larger than send (ratio 5:3) */
    NULL
};
```

### Do Not Block in Callbacks

```c
/* CRITICAL: Return promptly from all callbacks */
/* Blocking can deadlock the application */

solClient_rxMsgCallback_returnCode_t message_callback(
    solClient_opaqueSession_pt session_p,
    solClient_opaqueMsg_pt msg_p,
    void *user_p) {
    
    /* BAD: Blocking operations in callback */
    /* database_insert(msg_p);  // BLOCKS! */
    /* http_post(msg_p);        // BLOCKS! */
    
    /* GOOD: Queue message for worker thread */
    message_queue_push(msg_p);  /* Non-blocking */
    
    return SOLCLIENT_CALLBACK_OK;
}

/* Exception: Transacted sessions allow blocking in callback */
/* API creates message dispatcher thread for transacted sessions */
```

### Sending Best Practices

```c
/* Non-blocking send for better performance */
const char *sessionPropsNonBlock[] = {
    SOLCLIENT_SESSION_PROP_SEND_BLOCKING, SOLCLIENT_PROP_DISABLE_VAL,
    NULL
};

/* Handle SOLCLIENT_WOULD_BLOCK */
rc = solClient_session_sendMsg(session, msg);
if (rc == SOLCLIENT_WOULD_BLOCK) {
    /* Wait for SOLCLIENT_SESSION_EVENT_CAN_SEND event, then retry */
}

/* Batch sending for high throughput (up to 50 messages) */
solClient_opaqueMsg_pt batch[50];
/* Fill batch... */
solClient_session_sendMultipleMsg(session, batch, 50);

/* Set TTL on guaranteed messages to prevent queue buildup */
solClient_msg_setTimeToLive(msg, 60000);  /* 60 seconds */
```

### Flow Configuration

```c
/* AD Window Size should not exceed queue's max-delivered-unacked-msgs-per-flow */
const char *flowProps[] = {
    SOLCLIENT_FLOW_PROP_BIND_BLOCKING, SOLCLIENT_PROP_ENABLE_VAL,
    SOLCLIENT_FLOW_PROP_BIND_NAME, queueName,
    SOLCLIENT_FLOW_PROP_ACKMODE, SOLCLIENT_FLOW_PROP_ACKMODE_CLIENT,
    SOLCLIENT_FLOW_PROP_WINDOWSIZE, "25",  /* Match queue setting */
    NULL
};

/* For single-message processing, set window size to 1 */
/* Also configure queue: max-delivered-unacked-msgs-per-flow = 1 */

/* Size flows within broker's G-1 queue work units (default: 20,000)
 * Work unit = 2048 bytes
 * Formula: (num_flows * window_size * avg_msg_size) / 2048 < 20,000
 */
```

### File Descriptor Limits

```c
/* Linux/Solaris: Max 1023 sessions per process (select() limitation)
 * Windows: Max 63 sessions per context
 * 
 * If application manages its own FDs, this limit doesn't apply
 */
```

## Reference Links

- **C API Documentation**: https://docs.solace.com/API/Messaging-APIs/C-API/c-api-home.htm
- **API Reference**: https://docs.solace.com/API-Developer-Online-Ref-Documentation/c/index.html
- **Samples**: https://github.com/SolaceSamples/solace-samples-c

## Further Reading

- [Memory Optimization](./references/memory.md)
- [Performance Tuning](./references/performance.md)
- [Embedded Patterns](./references/embedded.md)
