# MQTT 5.0 Topic Aliases Implementation Guide

**Document Version:** 1.0
**Last Updated:** 2025-10-09
**Status:** Production Ready

---

## Table of Contents

1. [Overview](#overview)
2. [Business Value](#business-value)
3. [Technical Implementation](#technical-implementation)
4. [Topic Alias Mapping Strategy](#topic-alias-mapping-strategy)
5. [Client Implementation Examples](#client-implementation-examples)
6. [Server Configuration](#server-configuration)
7. [Error Handling](#error-handling)
8. [Testing and Validation](#testing-and-validation)
9. [Monitoring and Metrics](#monitoring-and-metrics)
10. [Bandwidth Savings Analysis](#bandwidth-savings-analysis)

---

## Overview

### What are Topic Aliases?

Topic aliases are an MQTT 5.0 feature that allows clients to replace long topic names with short 2-byte integer identifiers, dramatically reducing message overhead and cellular data costs.

**Standard MQTT Publish (without alias):**
```
Topic: "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/telemetry/vehicle" (54 bytes)
Payload: <protobuf data>
Total overhead: 54 bytes per message
```

**MQTT 5.0 Publish (with topic alias):**
```
Topic: "" (empty)
Topic Alias: 1 (2 bytes)
Payload: <protobuf data>
Total overhead: 2 bytes per message
```

**Bandwidth Savings: 96% reduction in topic overhead**

### How Topic Aliases Work

1. **Establishment Phase**: Client sends first message with full topic + alias
   ```
   PUBLISH {
     topic: "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/telemetry/vehicle",
     topicAlias: 1,
     payload: <data>
   }
   ```

2. **Subsequent Messages**: Client sends only alias
   ```
   PUBLISH {
     topic: "",  // Empty!
     topicAlias: 1,
     payload: <data>
   }
   ```

3. **Broker Translation**: MQTT broker maps alias→topic and routes message correctly

### Scope and Lifecycle

- **Per-connection**: Topic aliases are valid only for the specific MQTT connection
- **Directional**: Separate alias spaces for client→broker and broker→client
- **Reset on reconnect**: New connection requires re-establishing aliases
- **Maximum aliases**: Configurable (typically 10-100 per connection)

---

## Business Value

### Cost Savings

**Scenario: 1 million vehicles, each sending 10 telemetry messages per minute**

- **Messages per year**: 1M vehicles × 10 msg/min × 60 min × 24 hr × 365 days = **5.26 trillion messages**
- **Average topic length**: 54 bytes
- **Total topic overhead per year**: 5.26T × 54 bytes = **284 TB**
- **Cellular data cost**: $10 per GB = **$2.84 million per year**

**With Topic Aliases:**
- **Topic overhead per message**: 2 bytes (alias) + 54 bytes (first message only)
- **Effective topic overhead**: ~2.1 bytes per message (amortized)
- **Total topic overhead per year**: 5.26T × 2.1 bytes = **11 TB**
- **Savings**: 273 TB = **$2.73 million per year (96% reduction)**

### Additional Benefits

1. **Reduced cellular data usage** → lower bills for customers
2. **Faster message transmission** → lower latency
3. **Lower battery consumption** → longer vehicle range (EVs)
4. **Reduced AWS IoT Core costs** → lower infrastructure costs
5. **Better performance on poor networks** → improved reliability

---

## Technical Implementation

### AWS IoT Core Configuration

AWS IoT Core fully supports MQTT 5.0 topic aliases with these limits:

```yaml
# AWS IoT Core Topic Alias Configuration
max_topic_alias: 10              # Per connection (AWS default)
topic_alias_maximum: 65535       # Maximum alias value (MQTT 5.0 spec)
topic_alias_format: uint16       # 2-byte unsigned integer
```

**Note**: AWS IoT Core has a limit of **10 topic aliases per connection**. Choose the most frequently used topics.

### Topic Alias Assignment Strategy

**Priority Order (based on message frequency):**

| Alias | Topic Pattern                                         | Frequency | Priority |
|-------|------------------------------------------------------|-----------|----------|
| 1     | `v2c/v1/{region}/{vehicle_id}/telemetry/vehicle`     | Very High | Critical |
| 2     | `v2c/v1/{region}/{vehicle_id}/telemetry/batch`       | High      | Critical |
| 3     | `v2c/v1/{region}/{vehicle_id}/command/response`      | Medium    | High     |
| 4     | `v2c/v1/{region}/{vehicle_id}/diagnostics/dtc`       | Medium    | High     |
| 5     | `v2c/v1/{region}/{vehicle_id}/command/status`        | Medium    | Medium   |
| 6     | `v2c/v1/{region}/{vehicle_id}/diagnostics/health`    | Low       | Medium   |
| 7     | `v2c/v1/{region}/{vehicle_id}/ota/progress`          | Low       | Medium   |
| 8     | `v2c/v1/{region}/{vehicle_id}/ota/complete`          | Very Low  | Low      |
| 9     | `v2c/v1/{region}/{vehicle_id}/diagnostics/snapshot`  | Very Low  | Low      |
| 10    | `v2c/v1/{region}/{vehicle_id}/diagnostics/logs`      | Very Low  | Low      |

**Rationale**: Assign lower alias numbers to highest-frequency topics to maximize bandwidth savings.

---

## Topic Alias Mapping Strategy

### Vehicle-Side Implementation

```java
public class TopicAliasManager {
    // Maximum aliases supported by AWS IoT Core
    private static final int MAX_ALIASES = 10;

    // Pre-defined topic alias mapping
    private final Map<String, Integer> topicToAlias = new HashMap<>();
    private final Map<Integer, String> aliasToTopic = new HashMap<>();

    // Connection-specific state
    private final Set<Integer> establishedAliases = new HashSet<>();

    public TopicAliasManager(String region, String vehicleId) {
        // Initialize topic→alias mappings
        String prefix = "v2c/v1/" + region + "/" + vehicleId;

        topicToAlias.put(prefix + "/telemetry/vehicle", 1);
        topicToAlias.put(prefix + "/telemetry/batch", 2);
        topicToAlias.put(prefix + "/command/response", 3);
        topicToAlias.put(prefix + "/diagnostics/dtc", 4);
        topicToAlias.put(prefix + "/command/status", 5);
        topicToAlias.put(prefix + "/diagnostics/health", 6);
        topicToAlias.put(prefix + "/ota/progress", 7);
        topicToAlias.put(prefix + "/ota/complete", 8);
        topicToAlias.put(prefix + "/diagnostics/snapshot", 9);
        topicToAlias.put(prefix + "/diagnostics/logs", 10);

        // Create reverse mapping
        topicToAlias.forEach((topic, alias) -> aliasToTopic.put(alias, topic));
    }

    /**
     * Prepare message for publishing with topic alias
     */
    public MqttPublishMessage preparePublish(String topic, byte[] payload, int qos) {
        Integer alias = topicToAlias.get(topic);

        if (alias == null) {
            // Topic not configured for aliasing, send with full topic
            return new MqttPublishMessage(topic, null, payload, qos);
        }

        if (!establishedAliases.contains(alias)) {
            // First use of this alias, send full topic + alias
            establishedAliases.add(alias);
            return new MqttPublishMessage(topic, alias, payload, qos);
        }

        // Alias already established, send empty topic + alias
        return new MqttPublishMessage("", alias, payload, qos);
    }

    /**
     * Reset state on connection loss
     */
    public void onConnectionLost() {
        establishedAliases.clear();
    }

    /**
     * Incoming message processing (broker→vehicle)
     */
    public String resolveTopicAlias(String topic, Integer alias) {
        if (alias != null && topic.isEmpty()) {
            return aliasToTopic.get(alias);
        }
        return topic;
    }
}
```

### Cloud-Side Configuration

AWS IoT Core handles topic alias resolution automatically. No server-side code required.

**Configuration in IoT Core Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-*/telemetry/*",
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-*/command/*",
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-*/diagnostics/*",
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-*/ota/*"
      ]
    }
  ]
}
```

**Note**: Permissions are based on resolved topic (after alias translation), not the alias itself.

---

## Client Implementation Examples

### Java (Eclipse Paho MQTT 5.0)

```java
import org.eclipse.paho.mqttv5.client.MqttClient;
import org.eclipse.paho.mqttv5.common.MqttMessage;
import org.eclipse.paho.mqttv5.common.packet.MqttProperties;

public class V2CClient {
    private MqttClient client;
    private TopicAliasManager aliasManager;

    public void connect(String broker, String vehicleId) throws Exception {
        // Create MQTT 5.0 client
        client = new MqttClient(broker, vehicleId);

        MqttConnectionOptions options = new MqttConnectionOptions();
        options.setMqttVersion(MqttConnectionOptions.MQTT_VERSION_5_0);

        // Request topic alias maximum from server
        MqttProperties connectProperties = new MqttProperties();
        connectProperties.setTopicAliasMaximum(10);  // Request 10 aliases
        options.setConnectionProperties(connectProperties);

        client.connect(options);

        // Initialize alias manager
        aliasManager = new TopicAliasManager("us-east-1", vehicleId);
    }

    public void publishTelemetry(VehicleTelemetry telemetry) throws Exception {
        String topic = "v2c/v1/us-east-1/" + vehicleId + "/telemetry/vehicle";
        byte[] payload = telemetry.toByteArray();

        // Prepare message with topic alias
        MqttPublishMessage msg = aliasManager.preparePublish(topic, payload, 1);

        MqttMessage mqttMsg = new MqttMessage(msg.payload);
        mqttMsg.setQos(msg.qos);

        MqttProperties props = new MqttProperties();
        if (msg.topicAlias != null) {
            props.setTopicAlias(msg.topicAlias);
        }

        client.publish(msg.topic, mqttMsg, null, null, props);
    }
}
```

### C (Mosquitto MQTT 5.0)

```c
#include <mosquitto.h>

typedef struct {
    int topic_alias_map[11];  // 10 aliases + 1 (0 unused)
    bool alias_established[11];
    char* topics[11];
} topic_alias_manager_t;

void init_topic_aliases(topic_alias_manager_t* mgr, const char* region, const char* vehicle_id) {
    // Build topic strings
    char prefix[256];
    snprintf(prefix, sizeof(prefix), "v2c/v1/%s/%s", region, vehicle_id);

    // Alias 1: telemetry/vehicle
    asprintf(&mgr->topics[1], "%s/telemetry/vehicle", prefix);
    mgr->topic_alias_map[1] = 1;
    mgr->alias_established[1] = false;

    // Alias 2: telemetry/batch
    asprintf(&mgr->topics[2], "%s/telemetry/batch", prefix);
    mgr->topic_alias_map[2] = 2;
    mgr->alias_established[2] = false;

    // ... initialize remaining aliases
}

int publish_with_alias(struct mosquitto* mosq, topic_alias_manager_t* mgr,
                        int alias_num, const void* payload, int payload_len, int qos) {
    mosquitto_property* props = NULL;
    const char* topic;

    if (!mgr->alias_established[alias_num]) {
        // First use: send full topic + alias
        topic = mgr->topics[alias_num];
        mosquitto_property_add_int16(&props, MQTT_PROP_TOPIC_ALIAS, alias_num);
        mgr->alias_established[alias_num] = true;
    } else {
        // Subsequent use: empty topic + alias
        topic = "";
        mosquitto_property_add_int16(&props, MQTT_PROP_TOPIC_ALIAS, alias_num);
    }

    int rc = mosquitto_publish_v5(mosq, NULL, topic, payload_len, payload, qos, false, props);
    mosquitto_property_free_all(&props);

    return rc;
}

void on_disconnect(struct mosquitto* mosq, void* obj, int rc) {
    topic_alias_manager_t* mgr = (topic_alias_manager_t*)obj;

    // Reset alias establishment state
    for (int i = 1; i <= 10; i++) {
        mgr->alias_established[i] = false;
    }
}
```

### Python (paho-mqtt)

```python
import paho.mqtt.client as mqtt
from paho.mqtt.properties import Properties
from paho.mqtt.packettypes import PacketTypes

class TopicAliasManager:
    def __init__(self, region, vehicle_id):
        self.region = region
        self.vehicle_id = vehicle_id
        self.prefix = f"v2c/v1/{region}/{vehicle_id}"

        # Topic alias mappings
        self.topic_to_alias = {
            f"{self.prefix}/telemetry/vehicle": 1,
            f"{self.prefix}/telemetry/batch": 2,
            f"{self.prefix}/command/response": 3,
            # ... add remaining mappings
        }

        self.established_aliases = set()

    def prepare_publish(self, topic, payload, qos):
        """Prepare MQTT 5.0 publish with topic alias"""
        alias = self.topic_to_alias.get(topic)

        if alias is None:
            # No alias configured, use full topic
            return topic, payload, qos, None

        properties = Properties(PacketTypes.PUBLISH)
        properties.TopicAlias = alias

        if alias not in self.established_aliases:
            # First use: full topic + alias
            self.established_aliases.add(alias)
            return topic, payload, qos, properties
        else:
            # Subsequent use: empty topic + alias
            return "", payload, qos, properties

    def on_disconnect(self, client, userdata, rc):
        """Reset aliases on disconnect"""
        self.established_aliases.clear()

# Usage
client = mqtt.Client(protocol=mqtt.MQTTv5)
alias_mgr = TopicAliasManager("us-east-1", "VIN-1HGBH41JXMN109186")

client.on_disconnect = alias_mgr.on_disconnect

topic, payload, qos, props = alias_mgr.prepare_publish(
    "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/telemetry/vehicle",
    telemetry_data,
    1
)

client.publish(topic, payload, qos, properties=props)
```

---

## Server Configuration

### AWS IoT Core Settings

AWS IoT Core automatically supports topic aliases with no configuration required. The service negotiates topic alias maximum during CONNECT handshake.

**Connection Properties:**

```json
{
  "topicAliasMaximum": 10,
  "maximumPacketSize": 131072,
  "serverKeepAlive": 1200
}
```

### Monitoring Topic Alias Usage

AWS IoT Core CloudWatch metrics:

```yaml
Metrics:
  - MetricName: PublishIn.Success
    Dimensions:
      - Name: Protocol
        Value: MQTT
      - Name: ProtocolVersion
        Value: "5"
    Unit: Count

  - MetricName: BytesReceived
    Dimensions:
      - Name: Protocol
        Value: MQTT
    Unit: Bytes

  # Custom metric for bandwidth savings
  - MetricName: TopicAliasSavings
    Expression: (BaselineBytes - ActualBytes) / BaselineBytes * 100
    Unit: Percent
```

---

## Error Handling

### Common Errors and Solutions

#### 1. Topic Alias Not Established

**Error**: Broker rejects message with unknown alias

```
ERROR: Topic Alias 5 not established for this connection
MQTT Reason Code: 0x94 (Topic Alias Invalid)
```

**Solution**:
```java
// Always track alias establishment state
if (!establishedAliases.contains(alias)) {
    // Re-establish alias with full topic
    return new MqttPublishMessage(fullTopic, alias, payload, qos);
}
```

#### 2. Alias Limit Exceeded

**Error**: Client tries to use alias > maximum

```
ERROR: Topic Alias 11 exceeds maximum (10)
MQTT Reason Code: 0x94 (Topic Alias Invalid)
```

**Solution**:
```java
// Validate against negotiated maximum
if (alias > negotiatedMaximum) {
    // Fall back to full topic (no alias)
    return new MqttPublishMessage(fullTopic, null, payload, qos);
}
```

#### 3. Connection Reset Lost Aliases

**Error**: Using alias after reconnection

```
ERROR: Topic Alias 1 not valid (connection was reset)
```

**Solution**:
```java
@Override
public void onConnectionLost(Throwable cause) {
    aliasManager.reset();  // Clear all established aliases
}

@Override
public void onConnected() {
    // Aliases will be re-established on first use
}
```

#### 4. Broker Doesn't Support Aliases

**Error**: Broker returns `topicAliasMaximum = 0`

```
WARN: Broker does not support topic aliases
```

**Solution**:
```java
if (negotiatedMaximum == 0) {
    // Disable alias manager, use full topics
    aliasManager.setEnabled(false);
}
```

---

## Testing and Validation

### Unit Tests

```java
@Test
public void testTopicAliasEstablishment() {
    TopicAliasManager mgr = new TopicAliasManager("us-east-1", "VIN-TEST");
    String topic = "v2c/v1/us-east-1/VIN-TEST/telemetry/vehicle";

    // First publish: should include full topic
    MqttPublishMessage msg1 = mgr.preparePublish(topic, payload, 1);
    assertEquals(topic, msg1.topic);
    assertEquals(1, msg1.topicAlias.intValue());

    // Second publish: should use empty topic
    MqttPublishMessage msg2 = mgr.preparePublish(topic, payload, 1);
    assertEquals("", msg2.topic);
    assertEquals(1, msg2.topicAlias.intValue());
}

@Test
public void testConnectionResetClearsAliases() {
    TopicAliasManager mgr = new TopicAliasManager("us-east-1", "VIN-TEST");
    String topic = "v2c/v1/us-east-1/VIN-TEST/telemetry/vehicle";

    // Establish alias
    mgr.preparePublish(topic, payload, 1);

    // Simulate connection loss
    mgr.onConnectionLost();

    // Next publish should re-establish
    MqttPublishMessage msg = mgr.preparePublish(topic, payload, 1);
    assertEquals(topic, msg.topic);  // Full topic again
}
```

### Integration Tests

```python
def test_topic_alias_bandwidth_savings():
    """Verify bandwidth savings with topic aliases"""
    client = create_mqtt5_client()
    mgr = TopicAliasManager("us-east-1", "VIN-TEST")

    topic = "v2c/v1/us-east-1/VIN-TEST/telemetry/vehicle"
    payload = b"test_data"

    # Send 100 messages
    bytes_sent = 0
    for i in range(100):
        topic_to_send, _, _, props = mgr.prepare_publish(topic, payload, 1)

        # Track bytes sent
        bytes_sent += len(topic_to_send.encode('utf-8'))
        if props and props.TopicAlias:
            bytes_sent += 2  # 2 bytes for alias

        client.publish(topic_to_send, payload, properties=props)

    # Without aliases: 100 × 54 bytes = 5400 bytes
    # With aliases: 54 + (99 × 2) = 252 bytes
    expected_bytes = 54 + (99 * 2)
    assert bytes_sent == expected_bytes
    assert bytes_sent < 5400 * 0.1  # More than 90% savings
```

### AWS IoT Core Validation

```bash
# Monitor messages with CloudWatch Logs
aws logs tail /aws/iot/mqtt --follow --format short \
  --filter-pattern '{$.topicAlias = *}'

# Example output:
# {
#   "timestamp": "2025-10-09T10:30:00.000Z",
#   "clientId": "VIN-1HGBH41JXMN109186",
#   "topicAlias": 1,
#   "topic": "",  # Empty after first message
#   "payloadSize": 245
# }
```

---

## Monitoring and Metrics

### Key Performance Indicators

```yaml
# Prometheus metrics
vehicle_mqtt_topic_alias_established_total{alias="1"} 1500
vehicle_mqtt_topic_alias_established_total{alias="2"} 800

vehicle_mqtt_bytes_sent_total{with_alias="true"} 12000000
vehicle_mqtt_bytes_sent_total{with_alias="false"} 1500000

vehicle_mqtt_bandwidth_saved_bytes 42000000
vehicle_mqtt_bandwidth_saved_percent 96.3
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "V2C Topic Alias Performance",
    "panels": [
      {
        "title": "Bandwidth Savings (%)",
        "targets": [
          {
            "expr": "(vehicle_mqtt_bytes_baseline - vehicle_mqtt_bytes_sent_total{with_alias=\"true\"}) / vehicle_mqtt_bytes_baseline * 100"
          }
        ]
      },
      {
        "title": "Topic Alias Usage by Alias ID",
        "targets": [
          {
            "expr": "rate(vehicle_mqtt_topic_alias_established_total[5m])"
          }
        ]
      },
      {
        "title": "Messages Per Second (with vs without aliases)",
        "targets": [
          {
            "expr": "rate(vehicle_mqtt_messages_sent_total{with_alias=\"true\"}[1m])",
            "legendFormat": "With Aliases"
          },
          {
            "expr": "rate(vehicle_mqtt_messages_sent_total{with_alias=\"false\"}[1m])",
            "legendFormat": "Without Aliases"
          }
        ]
      }
    ]
  }
}
```

---

## Bandwidth Savings Analysis

### Detailed Calculation

**Assumptions:**
- 1 million vehicles
- 10 telemetry messages per minute per vehicle
- Average topic length: 54 bytes
- Message payload: 150 bytes (average)
- Operation: 24/7/365

**Without Topic Aliases:**

```
Topic overhead per message: 54 bytes
Messages per year: 1,000,000 × 10 × 60 × 24 × 365 = 5.26 trillion
Total topic overhead: 5.26T × 54 = 284 TB
Total payload: 5.26T × 150 = 789 TB
Total data: 1,073 TB = 1.05 PB
```

**With Topic Aliases:**

```
First message: 54 bytes (full topic) + 2 bytes (alias) = 56 bytes
Subsequent messages: 2 bytes (alias only)

Assuming 1 reconnection per day per vehicle:
- Reconnections per year: 1M × 365 = 365M
- First messages: 365M × 56 bytes = 20.4 GB
- Subsequent messages: (5.26T - 365M) × 2 bytes = 10.5 TB
- Total topic overhead: 10.52 TB

Total payload: 789 TB (unchanged)
Total data: 799.52 TB
```

**Savings:**

```
Topic overhead savings: 284 TB - 10.52 TB = 273.48 TB (96.3%)
Total data savings: 1,073 TB - 799.52 TB = 273.48 TB (25.5%)
Annual cost savings: 273.48 TB × $10/GB = $2.73 million
```

### Per-Vehicle Savings

```
Without aliases: 5.26 billion messages × 54 bytes = 284 MB/vehicle/year
With aliases: 5.26 billion messages × 2 bytes = 10.5 MB/vehicle/year
Savings: 273.5 MB/vehicle/year (96.3%)
```

### Cellular Data Cost Savings

```
Cellular data cost: $10 per GB (average global rate)
Per vehicle: 273.5 MB × $0.01/MB = $2.74/year
Fleet of 1M vehicles: $2.74M/year
```

### ROI Analysis

**Implementation Cost:**
- Development: 40 hours × $150/hr = $6,000
- Testing: 20 hours × $150/hr = $3,000
- Total one-time cost: $9,000

**Annual Savings:** $2.73 million

**ROI:** 30,233% or payback in 1.2 days

---

## Best Practices

### DO:
✅ Assign lower alias numbers to highest-frequency topics
✅ Reset alias state on connection loss
✅ Monitor bandwidth savings with metrics
✅ Use aliases for topics longer than 30 bytes
✅ Test alias behavior across reconnections
✅ Document alias assignments for maintenance

### DON'T:
❌ Exceed broker's `topicAliasMaximum`
❌ Use aliases for infrequent messages (overhead not worth it)
❌ Assume aliases persist across connections
❌ Hardcode alias numbers (use configuration)
❌ Forget to validate broker support (check CONNACK)
❌ Use dynamic alias assignment (causes confusion)

---

## Troubleshooting Guide

### Symptom: Messages Rejected with Error 0x94

**Cause**: Topic alias not established or invalid

**Fix**:
1. Check alias is within broker's maximum
2. Verify alias was established with full topic first
3. Reset aliases on reconnection
4. Add debug logging for alias state

### Symptom: No Bandwidth Savings Observed

**Cause**: Aliases not being used or incorrectly implemented

**Fix**:
1. Verify MQTT 5.0 protocol version in CONNECT
2. Check CONNACK for `topicAliasMaximum > 0`
3. Monitor network traffic to confirm empty topics
4. Validate alias establishment logic

### Symptom: Intermittent Message Delivery Failures

**Cause**: Alias state mismatch after reconnection

**Fix**:
1. Implement `onConnectionLost()` callback
2. Clear all alias state on disconnect
3. Re-establish aliases on first use after reconnect
4. Add retry logic for alias establishment failures

---

## References

- [MQTT 5.0 Specification - Topic Alias](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html#_Toc3901113)
- [AWS IoT Core MQTT 5.0 Support](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
- [Eclipse Paho MQTT 5.0 Java Client](https://github.com/eclipse/paho.mqtt.java)
- [Mosquitto MQTT 5.0 Library](https://mosquitto.org/man/libmosquitto-3.html)

---

**Document Maintained By:** Platform Engineering Team
**Review Cycle:** Quarterly
**Next Review:** 2025-12-31
