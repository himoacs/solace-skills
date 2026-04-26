---
name: solace-mqtt-development
description: 'Solace MQTT messaging guidance. Use for MQTT 3.1.1/5.0 protocol patterns, IoT device connectivity, QoS level selection, retained messages, last will testament, topic bridging between MQTT and SMF, and Paho client configuration.'
argument-hint: 'mqtt5, qos, retained, lwt, or iot'
---

# Solace MQTT Development

Expert guidance for MQTT messaging with Solace PubSub+ brokers.

## When to Use

- **IoT Applications**: Device-to-cloud messaging
- **Lightweight Clients**: Resource-constrained devices
- **MQTT 3.1.1/5.0**: Protocol-specific features
- **Interoperability**: Bridge MQTT with SMF/AMQP clients

## MQTT Support in Solace

| Feature | MQTT 3.1.1 | MQTT 5.0 |
|---------|------------|----------|
| QoS 0, 1, 2 | ✅ | ✅ |
| Retained Messages | ✅ | ✅ |
| Last Will Testament | ✅ | ✅ |
| Session Expiry | Limited | ✅ |
| Message Expiry | ❌ | ✅ |
| User Properties | ❌ | ✅ |
| Request/Response | Manual | ✅ Native |
| Shared Subscriptions | ✅ | ✅ |

## Connection

### Python (Paho)

```python
import paho.mqtt.client as mqtt
import ssl

def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        print("Connected successfully")
    else:
        print(f"Connection failed: {rc}")

def on_message(client, userdata, msg):
    print(f"Received: {msg.topic} -> {msg.payload.decode()}")

# MQTT 5.0 client
client = mqtt.Client(
    client_id="sensor-001",
    protocol=mqtt.MQTTv5,
    callback_api_version=mqtt.CallbackAPIVersion.VERSION2
)

client.on_connect = on_connect
client.on_message = on_message

# TLS configuration
client.tls_set(
    ca_certs="/path/to/ca.crt",
    certfile="/path/to/client.crt",
    keyfile="/path/to/client.key",
    tls_version=ssl.PROTOCOL_TLS
)

# Authentication
client.username_pw_set("mqtt-user", "password")

# Connect
client.connect(
    host="broker.example.com",
    port=8883,
    keepalive=60,
    clean_start=True
)

client.loop_start()
```

### JavaScript (MQTT.js)

```javascript
const mqtt = require('mqtt');

const client = mqtt.connect('wss://broker.example.com:8443', {
  clientId: 'web-client-001',
  username: 'mqtt-user',
  password: 'password',
  protocolVersion: 5,
  clean: true,
  keepalive: 60,
  reconnectPeriod: 5000,
  
  // MQTT 5.0 features
  properties: {
    sessionExpiryInterval: 3600,
    requestResponseInformation: true,
  }
});

client.on('connect', () => {
  console.log('Connected');
  client.subscribe('sensors/+/temperature', { qos: 1 });
});

client.on('message', (topic, payload, packet) => {
  console.log(`${topic}: ${payload.toString()}`);
  
  // MQTT 5.0 user properties
  if (packet.properties?.userProperties) {
    console.log('User properties:', packet.properties.userProperties);
  }
});
```

### Arduino/ESP32

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "WiFi-SSID";
const char* password = "WiFi-Password";
const char* mqtt_server = "broker.example.com";
const int mqtt_port = 1883;

WiFiClient espClient;
PubSubClient client(espClient);

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (unsigned int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Connecting MQTT...");
    if (client.connect("ESP32-001", "mqtt-user", "password")) {
      Serial.println("connected");
      client.subscribe("commands/esp32-001/#");
    } else {
      Serial.print("failed, rc=");
      Serial.println(client.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  
  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();
  
  // Publish telemetry
  char payload[50];
  snprintf(payload, 50, "{\"temp\": %.1f}", readTemperature());
  client.publish("sensors/esp32-001/temperature", payload);
  
  delay(10000);
}
```

## QoS Levels

### QoS Decision Matrix

| Scenario | Recommended QoS | Reason |
|----------|-----------------|--------|
| Telemetry streaming | QoS 0 | High volume, loss acceptable |
| Sensor readings | QoS 1 | At-least-once delivery |
| Commands | QoS 1 | Ensure delivery, idempotent |
| Financial transactions | QoS 2 | Exactly-once required |
| Configuration updates | QoS 1 + Retained | Persist for new clients |

### QoS Examples

```python
# QoS 0: Fire and forget
client.publish("telemetry/temp", "25.5", qos=0)

# QoS 1: At least once (with ACK)
result = client.publish("sensors/reading", payload, qos=1)
result.wait_for_publish()

# QoS 2: Exactly once
client.publish("payments/complete", order_json, qos=2)
```

## Retained Messages

```python
# Publish retained message (persists until replaced)
client.publish(
    "devices/thermostat/status",
    '{"online": true, "temp": 72}',
    qos=1,
    retain=True
)

# Clear retained message
client.publish("devices/thermostat/status", "", retain=True)

# Subscriber receives retained message immediately on subscribe
def on_message(client, userdata, msg):
    if msg.retain:
        print(f"Retained: {msg.topic}")
    print(msg.payload.decode())
```

## Last Will and Testament (LWT)

```python
# Set will before connecting
client.will_set(
    topic="devices/sensor-001/status",
    payload='{"online": false, "reason": "unexpected disconnect"}',
    qos=1,
    retain=True
)

client.connect(host, port)

# On graceful disconnect, clear the will manually
client.publish("devices/sensor-001/status", '{"online": false}', retain=True)
client.disconnect()
```

## MQTT 5.0 Features

### User Properties

```python
from paho.mqtt.properties import Properties
from paho.mqtt.packettypes import PacketTypes

# Publish with user properties
props = Properties(PacketTypes.PUBLISH)
props.UserProperty = [
    ("trace-id", "abc-123"),
    ("content-type", "application/json"),
    ("source", "temperature-sensor")
]

client.publish(
    "sensors/data",
    payload=json.dumps(data),
    qos=1,
    properties=props
)
```

### Request/Response

```python
# Requester
from uuid import uuid4

request_id = str(uuid4())
response_topic = f"responses/{client._client_id}/{request_id}"

# Subscribe to response topic first
client.subscribe(response_topic, qos=1)

# Send request
props = Properties(PacketTypes.PUBLISH)
props.ResponseTopic = response_topic
props.CorrelationData = request_id.encode()

client.publish(
    "services/calculator/add",
    json.dumps({"a": 5, "b": 3}),
    qos=1,
    properties=props
)

# Responder
def on_message(client, userdata, msg):
    if msg.topic.startswith("services/calculator/"):
        request = json.loads(msg.payload)
        result = request["a"] + request["b"]
        
        # Send response
        props = Properties(PacketTypes.PUBLISH)
        props.CorrelationData = msg.properties.CorrelationData
        
        client.publish(
            msg.properties.ResponseTopic,
            json.dumps({"result": result}),
            qos=1,
            properties=props
        )
```

### Session Expiry

```python
# MQTT 5.0 persistent session
connect_props = Properties(PacketTypes.CONNECT)
connect_props.SessionExpiryInterval = 3600  # 1 hour

client.connect(
    host, port,
    clean_start=False,
    properties=connect_props
)
```

### Message Expiry

```python
# Message expires after 60 seconds
props = Properties(PacketTypes.PUBLISH)
props.MessageExpiryInterval = 60

client.publish("alerts/temporary", "Check sensor", qos=1, properties=props)
```

## Shared Subscriptions

```python
# Multiple consumers share messages (load balancing)
# Format: $share/<group-name>/<topic-filter>

# Consumer 1
client1.subscribe("$share/workers/jobs/pending", qos=1)

# Consumer 2
client2.subscribe("$share/workers/jobs/pending", qos=1)

# Messages to jobs/pending are distributed across consumers
```

## Topic Mapping (MQTT to SMF)

MQTT topics map to Solace SMF topics:
- MQTT `/` separator maps to SMF `/`
- MQTT `+` wildcard maps to SMF `*`
- MQTT `#` wildcard maps to SMF `>`

| MQTT | Solace SMF |
|------|------------|
| `sensors/temp/device1` | `sensors/temp/device1` |
| `sensors/+/device1` | `sensors/*/device1` |
| `sensors/#` | `sensors/>` |

## Solace MQTT Configuration

### Enable MQTT Service (SEMP)

```bash
# Enable MQTT on default VPN
curl -X PATCH "http://localhost:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "serviceMqttPlainTextEnabled": true,
    "serviceMqttPlainTextListenPort": 1883,
    "serviceMqttTlsEnabled": true,
    "serviceMqttTlsListenPort": 8883,
    "serviceMqttWebSocketEnabled": true,
    "serviceMqttWebSocketListenPort": 8000
  }'
```

### MQTT Client Username Configuration

```bash
# Create MQTT-specific client username
curl -X POST "http://localhost:8080/SEMP/v2/config/msgVpns/default/clientUsernames" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "clientUsername": "mqtt-device",
    "enabled": true,
    "password": "secure-password"
  }'
```

## Best Practices

### Connection Management

```python
# Exponential backoff reconnection
class MqttClient:
    def __init__(self):
        self.client = mqtt.Client(client_id="device-001")
        self.client.on_disconnect = self.on_disconnect
        self.reconnect_delay = 1
        self.max_delay = 60
    
    def on_disconnect(self, client, userdata, rc):
        if rc != 0:
            print(f"Unexpected disconnect: {rc}")
            self.reconnect_with_backoff()
    
    def reconnect_with_backoff(self):
        while True:
            try:
                self.client.reconnect()
                self.reconnect_delay = 1  # Reset on success
                return
            except Exception as e:
                print(f"Reconnect failed, waiting {self.reconnect_delay}s")
                time.sleep(self.reconnect_delay)
                self.reconnect_delay = min(
                    self.reconnect_delay * 2,
                    self.max_delay
                )
```

### Efficient Payloads

```python
# Use compact JSON or binary for IoT
import struct

# Binary telemetry (10 bytes vs 30+ JSON)
def pack_telemetry(device_id, temp, humidity):
    return struct.pack('>HfH', device_id, temp, int(humidity * 100))

def unpack_telemetry(data):
    device_id, temp, humidity = struct.unpack('>HfH', data)
    return device_id, temp, humidity / 100
```

## Reference Links

- **MQTT Support**: https://docs.solace.com/MQTT/MQTT-Support.htm
- **MQTT 5.0 Features**: https://docs.solace.com/MQTT/MQTT-5-Features.htm
- **Topic Mapping**: https://docs.solace.com/MQTT/MQTT-Topic-Mapping.htm
- **Tutorials**: https://tutorials.solace.dev/mqtt

## Further Reading

- [QoS Selection](./references/qos-guide.md)
- [IoT Patterns](./references/iot-patterns.md)
- [Security](./references/mqtt-security.md)
