---
name: mqtt-patterns
description: Use when working with MQTT for IoT, mobile, or real-time applications — topic design, QoS levels, broker selection, device shadows, security, or MQTT 5.0 features
---

# MQTT Expert

## What MQTT Is

MQTT (ISO/IEC PRF 20922) is a publish-subscribe protocol over TCP designed for constrained devices and unreliable networks.

- 2-byte minimum fixed header — extremely low bandwidth
- Broker-centric: all clients connect to broker; broker routes by topic filter
- Persistent sessions: broker queues missed messages for offline clients
- Will message: broker publishes a configured message if client disconnects uncleanly
- MQTT 3.1.1: most widely supported; use as baseline
- MQTT 5.0: user properties, shared subscriptions, request/reply, message expiry, reason codes

```
      Device A                    Broker                   Backend
      ────────                  ──────────                 ───────
         │                          │                         │
         │── CONNECT ─────────────> │                         │
         │<─ CONNACK ───────────── │                         │
         │                          │                         │
         │── SUBSCRIBE              │                         │
         │   cmd/device-a/# ──────> │                         │
         │                          │<── SUBSCRIBE ─────────  │
         │                          │    dt/sensors/#         │
         │                          │                         │
         │── PUBLISH ─────────────> │                         │
         │   dt/sensors/.../temp    │── deliver ────────────> │
         │                          │                         │
         │<─ PUBLISH ────────────── │<── PUBLISH ──────────── │
         │   cmd/device-a/set/mode  │    cmd/device-a/set/mode│
```

---

## QoS Levels

| Level | Name | Delivery | Protocol overhead | Use case |
|-------|------|----------|-------------------|----------|
| 0 | At most once | Fire and forget; may lose | Smallest | High-frequency telemetry; loss acceptable |
| 1 | At least once | Guaranteed; may duplicate | Small | Most IoT events; handlers must be idempotent |
| 2 | Exactly once | Guaranteed; no duplicates | 4-way handshake; ~4x | Financial transactions; critical commands |

```
QoS 0:
  Publisher ──PUBLISH──> Broker ──PUBLISH──> Subscriber
  (no acknowledgment)

QoS 1:
  Publisher ──PUBLISH──> Broker ──PUBACK──> Publisher
  Broker ──PUBLISH──> Subscriber ──PUBACK──> Broker

QoS 2:
  Publisher ──PUBLISH──> Broker ──PUBREC──> Publisher ──PUBREL──> Broker ──PUBCOMP──> Publisher
  Broker ──PUBLISH──> Subscriber ──PUBREC──> Broker ──PUBREL──> Subscriber ──PUBCOMP──> Broker
```

**Default to QoS 1.** Design message handlers to be idempotent (deduplicate by device ID + timestamp/sequence). Use QoS 2 only when duplicates are genuinely unacceptable and you can afford the overhead.

---

## Topic Design Patterns

Topics are UTF-8 strings with `/` as a level separator. Design them hierarchically, from least to most specific.

```
# Wildcard reference
+    single level:  sensors/+/temperature  matches  sensors/zone1/temperature
#    multi-level:   sensors/#              matches  sensors/zone1/temperature
                                           and      sensors/zone1/zone2/temperature
     # must be the last character in the filter

# TELEMETRY — device to cloud (read data)
dt/{asset-type}/{location}/{device-id}/{metric}
dt/sensors/warehouse-a/sensor-001/temperature
dt/sensors/warehouse-a/sensor-001/humidity
dt/vehicles/fleet-west/truck-042/gps

# COMMANDS — cloud to device (write/actuate)
cmd/{device-id}/set/{property}
cmd/thermostat-042/set/target-temperature
cmd/pump-017/set/state

# COMMAND RESPONSE — device acknowledges command
cmd/{device-id}/response/{correlation-id}
cmd/thermostat-042/response/req-abc123

# DEVICE STATUS / HEALTH
dt/devices/{device-id}/status        # retained: online/offline
dt/devices/{device-id}/heartbeat     # periodic ping; no retain
dt/devices/{device-id}/errors

# DEVICE SHADOW / TWIN
$shadow/reported/{device-id}         # device publishes current state
$shadow/desired/{device-id}          # backend publishes target state
$shadow/delta/{device-id}            # optional: computed diff

# SYSTEM (reserved by broker — do not publish)
$SYS/#                               # broker stats, connections, uptime
```

---

## Python (paho-mqtt)

```python
import json
import ssl
import paho.mqtt.client as mqtt

BROKER = "broker.example.com"
PORT = 8883
CLIENT_ID = "backend-001"
TOPIC_SUBSCRIBE = "dt/sensors/+/+/temperature"
TOPIC_PUBLISH = "dt/sensors/warehouse-a/sensor-001/temperature"


def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        print("Connected")
        client.subscribe(TOPIC_SUBSCRIBE, qos=1)
    else:
        print(f"Connection failed: {rc}")


def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload)
        print(f"topic={msg.topic} value={payload['value']}")
        # idempotent processing: deduplicate by (device_id, timestamp)
        store_reading(msg.topic, payload)
    except Exception as exc:
        print(f"processing error: {exc}")


def on_disconnect(client, userdata, rc, properties=None):
    print(f"Disconnected rc={rc}")


# Will message — broker publishes if client disconnects uncleanly
will_payload = json.dumps({"status": "offline", "client": CLIENT_ID})

client = mqtt.Client(
    mqtt.CallbackAPIVersion.VERSION2,
    CLIENT_ID,
    clean_session=False,  # persistent session: broker queues missed QoS 1/2 messages
)
client.tls_set(tls_version=ssl.PROTOCOL_TLS_CLIENT)
client.username_pw_set("user", "password")
client.will_set("dt/devices/backend-001/status", will_payload, qos=1, retain=True)

client.on_connect = on_connect
client.on_message = on_message
client.on_disconnect = on_disconnect

client.connect(BROKER, PORT, keepalive=60)
client.loop_forever()


# Publish telemetry
client.publish(
    TOPIC_PUBLISH,
    json.dumps({"value": 23.5, "unit": "celsius", "ts": 1718000000}),
    qos=1,
    retain=False,
)
```

---

## Java (Eclipse Paho)

```java
import org.eclipse.paho.client.mqttv3.*;

MqttConnectOptions options = new MqttConnectOptions();
options.setUserName("user");
options.setPassword("password".toCharArray());
options.setCleanSession(false);          // persistent session
options.setKeepAliveInterval(60);
options.setAutomaticReconnect(true);
options.setMaxReconnectDelay(30_000);

// Will message — retained so new subscribers see "offline" if crash
options.setWill(
    "dt/devices/client-001/status",
    "{\"status\":\"offline\"}".getBytes(),
    1,     // QoS
    true   // retain
);

MqttClient client = new MqttClient(
    "ssl://broker.example.com:8883",
    "client-001",
    new MqttDefaultFilePersistence("/tmp/mqtt")  // persist QoS 1/2 across restarts
);

client.setCallback(new MqttCallback() {
    @Override
    public void connectionLost(Throwable cause) {
        System.err.println("Connection lost: " + cause.getMessage());
    }

    @Override
    public void messageArrived(String topic, MqttMessage message) throws Exception {
        System.out.printf("topic=%s payload=%s%n", topic, new String(message.getPayload()));
        processMessage(topic, message.getPayload());
    }

    @Override
    public void deliveryComplete(IMqttDeliveryToken token) {}
});

client.connect(options);
client.subscribe("cmd/client-001/#", 1);

// Publish telemetry
MqttMessage msg = new MqttMessage("{\"value\":23.5}".getBytes());
msg.setQos(1);
msg.setRetained(false);
client.publish("dt/sensors/warehouse-a/sensor-001/temperature", msg);

// Publish retained online status
MqttMessage status = new MqttMessage("{\"status\":\"online\"}".getBytes());
status.setQos(1);
status.setRetained(true);
client.publish("dt/devices/client-001/status", status);
```

---

## Go (paho.mqtt.golang)

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"

    mqtt "github.com/eclipse/paho.mqtt.golang"
)

func main() {
    opts := mqtt.NewClientOptions().
        AddBroker("ssl://broker.example.com:8883").
        SetClientID("go-client-001").
        SetUsername("user").
        SetPassword("password").
        SetCleanSession(false).
        SetKeepAlive(60 * time.Second).
        SetAutoReconnect(true).
        SetMaxReconnectInterval(30 * time.Second).
        SetWill(
            "dt/devices/go-client-001/status",
            `{"status":"offline"}`,
            1,
            true,
        ).
        SetOnConnectHandler(func(c mqtt.Client) {
            c.Subscribe("cmd/go-client-001/#", 1, commandHandler)
        }).
        SetConnectionLostHandler(func(c mqtt.Client, err error) {
            fmt.Printf("connection lost: %v\n", err)
        })

    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        panic(token.Error())
    }

    // Publish
    payload, _ := json.Marshal(map[string]any{"value": 23.5, "unit": "celsius"})
    token := client.Publish("dt/sensors/warehouse-a/sensor-001/temperature", 1, false, payload)
    token.Wait()
}

func commandHandler(client mqtt.Client, msg mqtt.Message) {
    fmt.Printf("command topic=%s payload=%s\n", msg.Topic(), string(msg.Payload()))
}
```

---

## Rust (rumqttc)

```rust
use rumqttc::{AsyncClient, Event, MqttOptions, Packet, QoS};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut mqttoptions = MqttOptions::new("rust-client-001", "broker.example.com", 8883);
    mqttoptions.set_credentials("user", "password");
    mqttoptions.set_keep_alive(Duration::from_secs(60));
    mqttoptions.set_clean_session(false); // persistent session

    // Will message
    mqttoptions.set_last_will(rumqttc::LastWill::new(
        "dt/devices/rust-client-001/status",
        r#"{"status":"offline"}"#,
        QoS::AtLeastOnce,
        true, // retain
    ));

    let (client, mut eventloop) = AsyncClient::new(mqttoptions, 32);

    client
        .subscribe("cmd/rust-client-001/#", QoS::AtLeastOnce)
        .await?;

    // Publish telemetry
    client
        .publish(
            "dt/sensors/warehouse-a/sensor-001/temperature",
            QoS::AtLeastOnce,
            false, // retain
            r#"{"value":23.5,"unit":"celsius"}"#,
        )
        .await?;

    loop {
        match eventloop.poll().await? {
            Event::Incoming(Packet::Publish(p)) => {
                println!("topic={} payload={:?}", p.topic, p.payload);
                // QoS 1: ack is automatic in rumqttc; client.ack() for manual mode
            }
            Event::Incoming(Packet::ConnAck(ack)) => {
                println!("connected: {:?}", ack.code);
            }
            _ => {}
        }
    }
}
```

---

## MQTT 5.0 Features

MQTT 5.0 adds first-class support for common enterprise patterns.

### User Properties — arbitrary headers on any packet

```python
# paho-mqtt 2.x with MQTT 5
from paho.mqtt.properties import Properties
from paho.mqtt.packettypes import PacketTypes

props = Properties(PacketTypes.PUBLISH)
props.UserProperty = [
    ("trace-id", "req-abc123"),
    ("tenant-id", "acme-corp"),
    ("schema-version", "2"),
]
client.publish(topic, payload, qos=1, properties=props)
```

### Message Expiry

```python
props = Properties(PacketTypes.PUBLISH)
props.MessageExpiryInterval = 300  # seconds; broker discards if undelivered after 5 min
client.publish(topic, payload, qos=1, properties=props)
```

### Request/Reply (native in MQTT 5.0)

```python
import uuid

# Requester
correlation_id = str(uuid.uuid4())
response_topic = f"_response/{CLIENT_ID}/{correlation_id}"
client.subscribe(response_topic, qos=1)

props = Properties(PacketTypes.PUBLISH)
props.ResponseTopic = response_topic
props.CorrelationData = correlation_id.encode()

client.publish("api/users/get", b'{"id":"123"}', qos=1, properties=props)

# Responder
def on_message(client, userdata, msg):
    if msg.properties.ResponseTopic:
        reply = get_user(msg.payload)
        rprops = Properties(PacketTypes.PUBLISH)
        rprops.CorrelationData = msg.properties.CorrelationData
        client.publish(msg.properties.ResponseTopic, reply, qos=1, properties=rprops)
```

### Shared Subscriptions — load balancing (MQTT 5.0 + some 3.x brokers)

```python
# Each message delivered to exactly ONE consumer in the group (round-robin or random)
GROUP = "processors"
FILTER = "dt/sensors/#"

# Three independent consumer processes:
client1.subscribe(f"$share/{GROUP}/{FILTER}", qos=1)
client2.subscribe(f"$share/{GROUP}/{FILTER}", qos=1)
client3.subscribe(f"$share/{GROUP}/{FILTER}", qos=1)
```

### Session Expiry

```python
from paho.mqtt.properties import Properties
from paho.mqtt.packettypes import PacketTypes

props = Properties(PacketTypes.CONNECT)
props.SessionExpiryInterval = 3600  # broker retains session for 1h after disconnect
client.connect(BROKER, PORT, keepalive=60, properties=props)
```

---

## Device Shadow / Twin Pattern

Device shadow decouples physical device state from the backend. Works even when the device is offline.

```
Backend publishes desired state
  PUBLISH $shadow/desired/thermostat-042
  retain=true QoS=1
  {"target_temperature": 22.0, "mode": "cooling"}

Device publishes reported state on connect and on change
  PUBLISH $shadow/reported/thermostat-042
  retain=true QoS=1
  {"temperature": 20.5, "mode": "heating", "ts": 1718000000}

Application-level reconciliation (device or backend computes delta):
  desired.target_temperature = 22.0
  reported.temperature       = 20.5
  → device should adjust

New backend subscriber receives current state immediately (retained messages)
```

**Key properties:**

- Both topics are retained — new subscribers get current state without waiting for next publish
- Device subscribes to `$shadow/desired/{id}` on connect; reads retained desired state
- Backend subscribes to `$shadow/reported/{id}`; gets latest device state on connect
- Device publishes to `$shadow/reported` periodically and on significant change

---

## Broker Selection

| Broker | Language | Connection scale | Best for |
|--------|----------|-----------------|----------|
| Mosquitto | C | ~100k | Edge, dev, small deployments |
| EMQX | Erlang | 10M+ | Enterprise IoT, cloud, rich rule engine |
| HiveMQ | Java | Millions | Enterprise, full MQTT 5.0, extensions |
| VerneMQ | Erlang | Large | Clustering, Kafka bridge |
| RobustMQ | Rust | Growing | Multi-protocol, Rust-native stack |
| AWS IoT Core | Managed | Serverless | AWS-native, device shadow, fleet indexing |
| Azure IoT Hub | Managed | Serverless | Azure-native, device twins, jobs |
| EMQX Cloud | Managed | Serverless | Hosted EMQX, multi-region |

**Decision guide:**

- On-prem, small fleet (<10k devices): Mosquitto
- On-prem, large fleet or enterprise: EMQX or HiveMQ
- AWS-native infrastructure: AWS IoT Core
- Azure-native infrastructure: Azure IoT Hub
- Rust stack or multi-protocol (MQTT + AMQP + Kafka): RobustMQ

---

## Security

```
Production security requirements:
  [x] TLS on port 8883 (never plain TCP 1883 in production)
  [x] Certificate validation; no InsecureSkipVerify
  [x] Device authentication via client certificates (mTLS) or username/password
  [x] ACLs: each device can only publish/subscribe to its own topics
  [x] Will message topic gated: only device X writes dt/devices/device-x/status
  [x] Separate credentials per device — no shared credentials across fleet
```

### mTLS device authentication

```python
client.tls_set(
    ca_certs="/etc/ssl/ca.crt",
    certfile="/etc/ssl/device-001.crt",
    keyfile="/etc/ssl/device-001.key",
    tls_version=ssl.PROTOCOL_TLS_CLIENT,
)
# No username/password needed — cert IS the identity
```

### ACL example (Mosquitto)

```
# mosquitto.acl
# Device can only publish telemetry and subscribe to its own commands
pattern readwrite dt/%u/#
pattern readwrite cmd/%u/#
pattern readwrite $shadow/reported/%u
pattern read     $shadow/desired/%u

# Backend service
user backend-svc
topic readwrite dt/#
topic readwrite cmd/#
topic readwrite $shadow/#
```

### Certificate pinning on devices

Pin the broker's public key fingerprint on the device firmware. Reject connections to any broker presenting a different certificate, even if CA-signed. Prevents DNS hijacking and rogue broker attacks.

---

## MQTT vs NATS vs AMQP vs Kafka

| Dimension | MQTT 3.1.1 | MQTT 5.0 | NATS Core | AMQP 1.0 | Kafka |
|-----------|-----------|---------|-----------|---------|-------|
| Protocol | TCP, WebSocket | TCP, WebSocket | TCP | TCP | TCP |
| Persistence | Broker-side session | Broker-side session | No | Broker queue | Yes (log) |
| Delivery | QoS 0/1/2 | QoS 0/1/2 | At-most-once | At-least-once | At-least-once |
| Wildcard routing | `+`, `#` | `+`, `#` | `*`, `>` | Exchange bindings | No |
| Consumer groups | No (shared sub: 5.0) | `$share/group/filter` | Queue groups | Competing consumers | Consumer groups |
| Headers/properties | No | User Properties | Headers (headers) | Application properties | Record headers |
| Binary overhead | 2-byte min | Slightly larger | Minimal | Larger | Larger |
| IoT suitability | Excellent | Excellent | Good | Moderate | Poor |

---

## Production Checklist

- [ ] TLS on port 8883; validate broker certificate against pinned CA
- [ ] Unique client ID per device; collision causes disconnect loops
- [ ] `clean_session=False` (MQTT 3) or `SessionExpiryInterval` (MQTT 5) for durable consumers
- [ ] Will message on each device: retained, QoS 1, to `dt/devices/{id}/status`
- [ ] Retained online status published after connect; cleared by Will on disconnect
- [ ] QoS 1 for commands and critical telemetry; QoS 0 acceptable for high-frequency low-value metrics
- [ ] Idempotent message handlers; deduplicate by device ID + sequence/timestamp
- [ ] Heartbeat topic with short-TTL retained message to detect stale devices
- [ ] ACLs in broker: each device scoped to its own topic subtree
- [ ] Separate credentials per device; rotate on revocation without re-flashing all devices
- [ ] Persistence store for client (Eclipse Paho Java MqttDefaultFilePersistence) to survive process restarts with QoS 1/2
- [ ] Monitor connected clients, subscriptions, message rate, and dropped connections
- [ ] MQTT 5.0 shared subscriptions for horizontally-scaled backend consumers

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Subscribing to `#` for all messages in a single consumer | Entire message volume to one process; cannot scale | Use specific topic filters or shared subscriptions (`$share/`) |
| QoS 2 for all messages | 4-way handshake; ~4x overhead vs QoS 1 | Use QoS 1; design handlers to be idempotent |
| Publishing to `$SYS/#` | Reserved for broker stats; publish rejected or silently dropped | Use your own namespace |
| No retained message for device status | New subscriber cannot tell if device is online until next heartbeat | Retain `{"status":"online"}` on connect; Will sets `{"status":"offline"}` |
| `cleanSession=True` for durable consumers | Missed messages while offline are lost | Use `cleanSession=False` (MQTT 3) or set `SessionExpiryInterval` (MQTT 5) |
| Shared client ID across multiple device instances | Broker disconnects previous client; reconnect loop between instances | One unique client ID per physical device or process |
| Plain TCP (port 1883) in production | All data and credentials in plaintext | Enforce TLS (port 8883) + cert pinning |
| No ACLs in broker | Any authenticated client can read/write any topic | Scope each client to minimum required topic set |
| Polling retained messages by reconnecting with `cleanSession=True` | Hammers broker with sessions; races with Will | Subscribe once; rely on retained messages delivered on subscribe |
| Unpinned certificate in device firmware | Susceptible to rogue broker (DNS hijack, MITM) | Pin CA cert or broker cert fingerprint in firmware |

---

## Observability

### Mosquitto built-in stats (subscribe to `$SYS/#`)

```bash
mosquitto_sub -h broker -t '$SYS/#' -v
# $SYS/broker/clients/connected  42
# $SYS/broker/messages/received  190234
# $SYS/broker/messages/sent      187001
```

### EMQX Dashboard and API

```bash
# Connected clients
curl http://emqx:18083/api/v5/clients -u admin:public | jq '.data | length'

# Subscriptions
curl http://emqx:18083/api/v5/subscriptions | jq .

# Message rates
curl http://emqx:18083/api/v5/metrics | jq '.messages_received'
```

### Key metrics to alert on

| Metric | Alert threshold (example) |
|--------|--------------------------|
| `clients/connected` drops sharply | >10% drop in 1 min |
| Subscription count spikes | 2x baseline in 5 min |
| QoS 1/2 message redelivery rate | >1% |
| Client disconnect rate | >5/min |
| Message queue depth per session | >1000 (slow consumer) |
