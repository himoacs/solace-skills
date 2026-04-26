---
name: solace-javascript-development
description: 'Solace JavaScript/TypeScript development guidance. Use for solclientjs browser and Node.js patterns, WebSocket connections, CORS configuration, TypeScript types, React/Angular/Vue integration, and bundler setup (webpack, vite).'
argument-hint: 'browser, nodejs, typescript, react, angular, or vue'
---

# Solace JavaScript Development

Expert guidance for building JavaScript/TypeScript applications with Solace PubSub+.

## When to Use

- **Browser Applications**: Web clients with WebSocket
- **Node.js**: Server-side JavaScript applications
- **TypeScript**: Type-safe Solace development
- **Frontend Frameworks**: React, Angular, Vue integration
- **Bundler Setup**: webpack, vite configuration

## Installation

```bash
# npm
npm install solclientjs

# yarn
yarn add solclientjs
```

## Browser Setup

### Basic HTML

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/solclientjs@10.x/lib/solclient.js"></script>
</head>
<body>
  <script>
    const solace = window.solace;
    // Use solace API
  </script>
</body>
</html>
```

### ES Module Import

```javascript
import * as solace from 'solclientjs';

// Initialize factory
const factoryProps = new solace.SolclientFactoryProperties();
factoryProps.profile = solace.SolclientFactoryProfiles.version10;
solace.SolclientFactory.init(factoryProps);
```

## Connection

### Basic Connection

```javascript
class SolaceClient {
  constructor() {
    this.session = null;
    this.connected = false;
  }

  connect(config) {
    return new Promise((resolve, reject) => {
      const sessionProperties = {
        url: config.url,           // 'ws://localhost:8008' or 'wss://...'
        vpnName: config.vpn,
        userName: config.username,
        password: config.password,
        
        // Reconnection
        reconnectRetries: -1,      // Infinite
        reconnectRetryWaitInMsecs: 3000,
        reapplySubscriptions: true,
        
        // Timeouts
        connectTimeoutInMsecs: 10000,
        readTimeoutInMsecs: 10000,
      };

      this.session = solace.SolclientFactory.createSession(sessionProperties);

      // Event handlers
      this.session.on(solace.SessionEventCode.UP_NOTICE, () => {
        this.connected = true;
        resolve();
      });

      this.session.on(solace.SessionEventCode.CONNECT_FAILED_ERROR, (event) => {
        reject(new Error(`Connect failed: ${event.infoStr}`));
      });

      this.session.on(solace.SessionEventCode.DISCONNECTED, () => {
        this.connected = false;
        console.log('Disconnected');
      });

      this.session.on(solace.SessionEventCode.RECONNECTING, () => {
        console.log('Reconnecting...');
      });

      this.session.on(solace.SessionEventCode.RECONNECTED, () => {
        this.connected = true;
        console.log('Reconnected!');
      });

      try {
        this.session.connect();
      } catch (error) {
        reject(error);
      }
    });
  }

  disconnect() {
    if (this.session) {
      this.session.disconnect();
    }
  }
}
```

### WebSocket URLs

```javascript
// Development (non-TLS)
const devUrl = 'ws://localhost:8008';

// Production (TLS)
const prodUrl = 'wss://mr-connection-xxx.messaging.solace.cloud:443';

// Multiple hosts for failover
const haUrl = 'wss://primary:443,wss://backup:443';
```

## Publishing

### Direct Messages

```javascript
class Publisher {
  constructor(session) {
    this.session = session;
  }

  publish(topicName, payload) {
    const topic = solace.SolclientFactory.createTopicDestination(topicName);
    const message = solace.SolclientFactory.createMessage();
    
    if (typeof payload === 'object') {
      message.setBinaryAttachment(JSON.stringify(payload));
      message.setUserPropertyMap({ 
        contentType: 'application/json' 
      });
    } else {
      message.setBinaryAttachment(payload);
    }
    
    message.setDeliveryMode(solace.MessageDeliveryModeType.DIRECT);
    message.setDestination(topic);
    
    this.session.send(message);
  }

  publishWithCorrelation(topicName, payload, correlationId) {
    const message = solace.SolclientFactory.createMessage();
    message.setBinaryAttachment(JSON.stringify(payload));
    message.setCorrelationId(correlationId);
    message.setDestination(
      solace.SolclientFactory.createTopicDestination(topicName)
    );
    
    this.session.send(message);
  }
}
```

### Guaranteed Messages

```javascript
class GuaranteedPublisher {
  constructor(session) {
    this.session = session;
    this.correlationMap = new Map();
  }

  async publish(destination, payload, options = {}) {
    return new Promise((resolve, reject) => {
      const correlationKey = crypto.randomUUID();
      
      this.correlationMap.set(correlationKey, { resolve, reject });
      
      const message = solace.SolclientFactory.createMessage();
      message.setBinaryAttachment(JSON.stringify(payload));
      message.setDeliveryMode(solace.MessageDeliveryModeType.PERSISTENT);
      message.setCorrelationKey(correlationKey);
      
      if (options.queue) {
        message.setDestination(
          solace.SolclientFactory.createDurableQueueDestination(destination)
        );
      } else {
        message.setDestination(
          solace.SolclientFactory.createTopicDestination(destination)
        );
      }

      try {
        this.session.send(message);
      } catch (error) {
        this.correlationMap.delete(correlationKey);
        reject(error);
      }
    });
  }

  setupAckHandler() {
    this.session.on(solace.SessionEventCode.ACKNOWLEDGED_MESSAGE, (event) => {
      const entry = this.correlationMap.get(event.correlationKey);
      if (entry) {
        entry.resolve();
        this.correlationMap.delete(event.correlationKey);
      }
    });

    this.session.on(solace.SessionEventCode.REJECTED_MESSAGE_ERROR, (event) => {
      const entry = this.correlationMap.get(event.correlationKey);
      if (entry) {
        entry.reject(new Error(event.infoStr));
        this.correlationMap.delete(event.correlationKey);
      }
    });
  }
}
```

## Subscribing

### Topic Subscription

```javascript
class Subscriber {
  constructor(session) {
    this.session = session;
    this.handlers = new Map();
  }

  subscribe(topicPattern, handler) {
    const topic = solace.SolclientFactory.createTopicDestination(topicPattern);
    
    this.handlers.set(topicPattern, handler);
    
    this.session.subscribe(
      topic,
      true,  // requestConfirmation
      topicPattern,  // correlationKey
      10000  // timeout
    );
  }

  setupMessageHandler() {
    this.session.on(solace.SessionEventCode.MESSAGE, (message) => {
      const topic = message.getDestination().getName();
      const payload = message.getBinaryAttachment();
      
      // Parse JSON if applicable
      let data = payload;
      try {
        data = JSON.parse(payload);
      } catch (e) {
        // Not JSON, use raw payload
      }

      // Find matching handler
      for (const [pattern, handler] of this.handlers) {
        if (this.matchesTopic(topic, pattern)) {
          handler(data, {
            topic,
            correlationId: message.getCorrelationId(),
            timestamp: message.getSenderTimestamp(),
          });
        }
      }
    });
  }

  matchesTopic(topic, pattern) {
    // Simple wildcard matching (* and >)
    const regex = pattern
      .replace(/\*/g, '[^/]+')
      .replace(/>/g, '.*');
    return new RegExp(`^${regex}$`).test(topic);
  }
}
```

### Queue Consumer

```javascript
class QueueConsumer {
  constructor(session) {
    this.session = session;
    this.messageConsumer = null;
  }

  consume(queueName, handler) {
    const queue = solace.SolclientFactory.createDurableQueueDestination(queueName);
    
    const consumerProperties = {
      queueDescriptor: { name: queueName, type: solace.QueueType.QUEUE },
      acknowledgeMode: solace.MessageConsumerAcknowledgeMode.CLIENT,
    };

    this.messageConsumer = this.session.createMessageConsumer(consumerProperties);

    this.messageConsumer.on(solace.MessageConsumerEventName.MESSAGE, (message) => {
      const payload = JSON.parse(message.getBinaryAttachment());
      
      try {
        handler(payload, message);
        message.acknowledge();
      } catch (error) {
        console.error('Processing failed:', error);
        // Message will be redelivered
      }
    });

    this.messageConsumer.on(solace.MessageConsumerEventName.UP, () => {
      console.log(`Connected to queue: ${queueName}`);
    });

    this.messageConsumer.connect();
  }

  disconnect() {
    if (this.messageConsumer) {
      this.messageConsumer.disconnect();
    }
  }
}
```

## TypeScript Support

### Type Definitions

```typescript
// solace-types.ts
import * as solace from 'solclientjs';

export interface SolaceConfig {
  url: string;
  vpn: string;
  username: string;
  password: string;
}

export interface MessageMetadata {
  topic: string;
  correlationId?: string;
  timestamp?: number;
  userProperties?: Record<string, unknown>;
}

export type MessageHandler<T = unknown> = (
  data: T,
  metadata: MessageMetadata
) => void | Promise<void>;

export interface PublishOptions {
  correlationId?: string;
  ttl?: number;
  dmqEligible?: boolean;
  priority?: number;
}
```

### Typed Client

```typescript
// SolaceClient.ts
import * as solace from 'solclientjs';
import { SolaceConfig, MessageHandler, PublishOptions } from './solace-types';

export class TypedSolaceClient {
  private session: solace.Session | null = null;
  
  async connect(config: SolaceConfig): Promise<void> {
    // ... connection logic
  }

  publish<T>(topic: string, payload: T, options?: PublishOptions): void {
    if (!this.session) throw new Error('Not connected');
    
    const message = solace.SolclientFactory.createMessage();
    message.setBinaryAttachment(JSON.stringify(payload));
    message.setDestination(
      solace.SolclientFactory.createTopicDestination(topic)
    );
    
    if (options?.correlationId) {
      message.setCorrelationId(options.correlationId);
    }
    
    this.session.send(message);
  }

  subscribe<T>(pattern: string, handler: MessageHandler<T>): void {
    // ... subscription logic with typed handler
  }
}
```

## React Integration

### Custom Hook

```typescript
// useSolace.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import * as solace from 'solclientjs';

export function useSolace(config: SolaceConfig) {
  const [connected, setConnected] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const sessionRef = useRef<solace.Session | null>(null);

  useEffect(() => {
    const connect = async () => {
      try {
        // Initialize and connect
        const session = solace.SolclientFactory.createSession({
          url: config.url,
          vpnName: config.vpn,
          userName: config.username,
          password: config.password,
        });

        session.on(solace.SessionEventCode.UP_NOTICE, () => {
          setConnected(true);
        });

        session.on(solace.SessionEventCode.DISCONNECTED, () => {
          setConnected(false);
        });

        session.connect();
        sessionRef.current = session;
      } catch (e) {
        setError(e as Error);
      }
    };

    connect();

    return () => {
      sessionRef.current?.disconnect();
    };
  }, [config.url, config.vpn, config.username, config.password]);

  const publish = useCallback((topic: string, payload: unknown) => {
    if (!sessionRef.current) return;
    
    const message = solace.SolclientFactory.createMessage();
    message.setBinaryAttachment(JSON.stringify(payload));
    message.setDestination(
      solace.SolclientFactory.createTopicDestination(topic)
    );
    sessionRef.current.send(message);
  }, []);

  const subscribe = useCallback((pattern: string, handler: (data: unknown) => void) => {
    if (!sessionRef.current) return;

    sessionRef.current.on(solace.SessionEventCode.MESSAGE, (message) => {
      const payload = JSON.parse(message.getBinaryAttachment());
      handler(payload);
    });

    sessionRef.current.subscribe(
      solace.SolclientFactory.createTopicDestination(pattern),
      true,
      pattern,
      10000
    );
  }, []);

  return { connected, error, publish, subscribe };
}
```

### React Component

```tsx
// OrderDashboard.tsx
import React, { useEffect, useState } from 'react';
import { useSolace } from './useSolace';

interface Order {
  id: string;
  status: string;
  amount: number;
}

export function OrderDashboard() {
  const [orders, setOrders] = useState<Order[]>([]);
  const { connected, subscribe, publish } = useSolace({
    url: 'wss://broker:443',
    vpn: 'default',
    username: 'user',
    password: 'pass',
  });

  useEffect(() => {
    if (connected) {
      subscribe('orders/created', (order: Order) => {
        setOrders(prev => [...prev, order]);
      });
    }
  }, [connected, subscribe]);

  const handleNewOrder = () => {
    publish('orders/new', {
      id: crypto.randomUUID(),
      status: 'pending',
      amount: 99.99,
    });
  };

  return (
    <div>
      <h1>Orders {connected ? '🔵' : '🔴'}</h1>
      <button onClick={handleNewOrder}>New Order</button>
      <ul>
        {orders.map(order => (
          <li key={order.id}>{order.id}: ${order.amount}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Bundler Configuration

### Webpack

```javascript
// webpack.config.js
module.exports = {
  resolve: {
    fallback: {
      // solclientjs may need these polyfills
      "buffer": require.resolve("buffer/"),
      "stream": require.resolve("stream-browserify"),
    }
  },
  plugins: [
    new webpack.ProvidePlugin({
      Buffer: ['buffer', 'Buffer'],
    }),
  ],
};
```

### Vite

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { nodePolyfills } from 'vite-plugin-node-polyfills';

export default defineConfig({
  plugins: [
    nodePolyfills({
      include: ['buffer', 'stream'],
    }),
  ],
  optimizeDeps: {
    include: ['solclientjs'],
  },
});
```

## CORS Configuration

For browser clients connecting to Solace Cloud or on-premise brokers:

```
# Solace Cloud: CORS is pre-configured

# Software Broker: Enable CORS via SEMP
curl -X PATCH \
  "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -d '{"webTransportCorsAllowedOrigin": "*"}'
```

## Reference Links

- **JavaScript API Documentation**: https://docs.solace.com/API/Messaging-APIs/JavaScript-API/js-home.htm
- **Node.js API**: https://docs.solace.com/API/Messaging-APIs/NodeJS-API/node-js-home.htm
- **Tutorials**: https://tutorials.solace.dev/javascript
- **GitHub Samples**: https://github.com/SolaceSamples/solace-samples-javascript

## Further Reading

- [Browser Security](./references/browser-security.md)
- [React Patterns](./references/react-patterns.md)
- [TypeScript Examples](./references/typescript.md)
