# MQTT QoS Selection Guide

**Version:** 1.0
**Status:** Draft
**Last Updated:** 2025-10-09
**Owner:** Vehicle Connectivity Architecture Team
**Standards Compliance:** MQTT 5.0 Specification, ISO 21434

---

## Table of Contents

1. [Overview](#overview)
2. [MQTT QoS Levels Explained](#mqtt-qos-levels-explained)
3. [QoS Selection Decision Tree](#qos-selection-decision-tree)
4. [Message Type to QoS Mapping](#message-type-to-qos-mapping)
5. [Performance and Cost Implications](#performance-and-cost-implications)
6. [Idempotency Requirements](#idempotency-requirements)
7. [Best Practices](#best-practices)
8. [Testing and Validation](#testing-and-validation)
9. [Examples](#examples)
10. [References](#references)

---

## Overview

Quality of Service (QoS) is a critical design decision in MQTT-based vehicle-to-cloud communications. The QoS level determines the delivery guarantee for each message, directly impacting:

- **Reliability:** Likelihood of message delivery
- **Latency:** Time to acknowledge delivery
- **Bandwidth:** Network data consumption
- **Cost:** Cellular data charges
- **Battery:** Vehicle power consumption

This guide provides systematic guidance for selecting the appropriate QoS level for each message type in the V2C architecture.

### Design Principles

1. **Default to QoS 1:** Most messages should use QoS 1 (at-least-once delivery)
2. **Reserve QoS 2 for Critical:** Only use QoS 2 for safety-critical or financially sensitive operations
3. **Use QoS 0 Sparingly:** Limit QoS 0 to high-frequency, non-essential telemetry
4. **Consider Idempotency:** All messages should be designed to handle duplicate delivery
5. **Measure and Optimize:** Monitor delivery rates and adjust QoS based on real-world data

---

## MQTT QoS Levels Explained

### QoS 0: At Most Once

**Delivery Guarantee:** Fire-and-forget, no acknowledgment

**Protocol Flow:**
```
Publisher → [PUBLISH] → Broker → [PUBLISH] → Subscriber
            (no ACK)              (no ACK)
```

**Characteristics:**
- **Reliability:** No guarantee of delivery
- **Latency:** Lowest (~5-50ms)
- **Bandwidth:** Minimal (message size only)
- **Duplicates:** None (message sent once)
- **Ordering:** Not guaranteed

**When to Use:**
- High-frequency telemetry (>1 message/second)
- Redundant data where loss is acceptable
- Real-time status updates (superseded by newer data)

**Examples:**
- GPS location updates (every 30 seconds)
- Battery temperature (every 60 seconds)
- Vehicle speed (every 5 seconds)

---

### QoS 1: At Least Once

**Delivery Guarantee:** Message delivered at least once, may have duplicates

**Protocol Flow:**
```
Publisher → [PUBLISH] → Broker → [PUBLISH] → Subscriber
          ← [PUBACK]            ← [PUBACK]
```

**Characteristics:**
- **Reliability:** Guaranteed delivery
- **Latency:** Medium (~50-200ms, includes round-trip ACK)
- **Bandwidth:** 2x (message + acknowledgment)
- **Duplicates:** Possible (network timeouts, retries)
- **Ordering:** Not guaranteed (use sequence numbers if needed)

**When to Use:**
- Important telemetry (alerts, events)
- Most remote commands
- Diagnostic data collection
- Default for most V2C messages

**Examples:**
- Door lock/unlock commands
- Battery low warning
- DTC (diagnostic trouble code) alerts
- Climate control commands

---

### QoS 2: Exactly Once

**Delivery Guarantee:** Message delivered exactly once, no duplicates

**Protocol Flow:**
```
Publisher → [PUBLISH] → Broker → [PUBLISH] → Subscriber
          ← [PUBREC]            ← [PUBREC]
          → [PUBREL]            → [PUBREL]
          ← [PUBCOMP]           ← [PUBCOMP]
```

**Characteristics:**
- **Reliability:** Guaranteed delivery, exactly once
- **Latency:** Highest (~200-500ms, 4-way handshake)
- **Bandwidth:** 4x (message + 3 acknowledgments)
- **Duplicates:** None (protocol prevents)
- **Ordering:** Not guaranteed

**When to Use:**
- Safety-critical commands (emergency brake, airbag deactivation)
- Financial transactions (billing events, tolls)
- OTA update orchestration
- Non-idempotent operations

**Examples:**
- Emergency stop command
- Billing event (tollbooth transaction)
- OTA firmware install trigger
- Remote valet mode activation

---

## QoS Selection Decision Tree

```
START: Does the message affect safety or lives?
  ├─ YES → Use QoS 2 (Exactly Once)
  │        Examples: Emergency stop, airbag control
  │
  └─ NO → Does the message have financial impact?
       ├─ YES → Use QoS 2 (Exactly Once)
       │        Examples: Billing events, payment authorization
       │
       └─ NO → Is message loss acceptable?
            ├─ YES → Is the message sent frequently (>1/sec)?
            │   ├─ YES → Use QoS 0 (At Most Once)
            │   │        Examples: Real-time speed, periodic GPS
            │   │
            │   └─ NO → Use QoS 1 (At Least Once)
            │            Examples: Battery warnings, alerts
            │
            └─ NO → Is the operation idempotent?
                 ├─ YES → Use QoS 1 (At Least Once)
                 │        Examples: Lock doors, start climate
                 │        Note: Use idempotency keys
                 │
                 └─ NO → Use QoS 2 (Exactly Once)
                          Examples: Account debit, one-time config
```

---

## Message Type to QoS Mapping

### Telemetry Messages (Vehicle → Cloud)

| Message Type              | QoS Level | Frequency    | Rationale                                    |
|---------------------------|-----------|--------------|----------------------------------------------|
| GPS Location              | 0         | 30-60s       | High frequency, superseded by newer data     |
| Battery SoC               | 0         | 60s          | Periodic, loss acceptable                    |
| Cabin Temperature         | 0         | 60s          | Non-critical, redundant                      |
| Vehicle Speed             | 0         | 5s           | High frequency, real-time only               |
| DTC Alert                 | 1         | Event-driven | Important diagnostic info                    |
| Battery Low Warning       | 1         | Event-driven | User expects notification                    |
| Crash Detection           | 2         | Event-driven | Safety-critical, triggers emergency response |
| Airbag Deployment         | 2         | Event-driven | Legal/insurance requirement                  |
| Toll Transaction          | 2         | Event-driven | Financial impact                             |
| OTA Download Complete     | 1         | Event-driven | Installation depends on this                 |
| OTA Install Status        | 1         | Event-driven | Progress tracking                            |

---

### Command Messages (Cloud → Vehicle)

| Command Type              | QoS Level | Idempotent? | Rationale                                    |
|---------------------------|-----------|-------------|----------------------------------------------|
| Lock Doors                | 1         | Yes         | Safe to repeat, uses idempotency key         |
| Unlock Doors              | 1         | Yes         | Safe to repeat                               |
| Start Climate             | 1         | Yes         | Safe to repeat (check state first)           |
| Horn/Lights               | 1         | Yes         | Safe to repeat, short duration               |
| Remote Engine Start       | 1         | Yes*        | Check engine state before starting           |
| Charge Start/Stop (EV)    | 1         | Yes         | Safe to repeat, stateful operation           |
| OTA Update Available      | 1         | Yes         | Notification only, no action                 |
| OTA Download Start        | 1         | Yes         | Resumable download                           |
| OTA Install               | 2         | No          | One-time operation, cannot be reversed easily|
| Emergency Stop            | 2         | No          | Safety-critical, exactly once required       |
| Valet Mode Activate       | 2         | No          | Configuration change, speed/distance limits  |
| Disable Airbag            | 2         | No          | Safety-critical configuration                |
| Factory Reset             | 2         | No          | Destructive operation                        |

\* *Safe to repeat IF vehicle checks engine state before executing*

---

### OTA Update Messages

| Message                   | QoS | Direction       | Rationale                              |
|---------------------------|-----|-----------------|----------------------------------------|
| OTA Available             | 1   | Cloud → Vehicle | Notification, repeatable               |
| OTA Accept                | 1   | Vehicle → Cloud | User consent, idempotent               |
| OTA Download Progress     | 0   | Vehicle → Cloud | High frequency, superseded             |
| OTA Download Complete     | 1   | Vehicle → Cloud | Important milestone                    |
| OTA Install Start         | 2   | Cloud → Vehicle | Trigger for irreversible operation     |
| OTA Install Progress      | 1   | Vehicle → Cloud | Status update                          |
| OTA Install Complete      | 2   | Vehicle → Cloud | Critical success confirmation          |
| OTA Rollback              | 2   | Cloud → Vehicle | Safety-critical recovery               |

---

## Performance and Cost Implications

### Bandwidth Comparison

**Scenario:** 100,000 vehicles sending GPS location every 60 seconds

**Message Size:** 100 bytes (protobuf payload + MQTT headers)

| QoS Level | Messages/Day | Bandwidth/Message | Total Bandwidth/Day |
|-----------|--------------|-------------------|---------------------|
| QoS 0     | 144M         | 100 bytes         | 14.4 GB             |
| QoS 1     | 144M         | 200 bytes (2x)    | 28.8 GB             |
| QoS 2     | 144M         | 400 bytes (4x)    | 57.6 GB             |

**Annual Bandwidth:**
- QoS 0: 5.3 TB/year
- QoS 1: 10.5 TB/year
- QoS 2: 21.0 TB/year

**Cost Implications (at $0.10/GB):**
- QoS 0: $525/year
- QoS 1: $1,050/year
- QoS 2: $2,100/year

**Per-Vehicle Annual Cost:**
- QoS 0: $0.0053
- QoS 1: $0.0105
- QoS 2: $0.0210

---

### Latency Comparison

**Network Conditions:** 100ms RTT (round-trip time), reliable connection

| QoS Level | Latency      | Explanation                              |
|-----------|--------------|------------------------------------------|
| QoS 0     | ~5-50ms      | Direct publish, no acknowledgment        |
| QoS 1     | ~100-200ms   | Publish + PUBACK (1 round-trip)          |
| QoS 2     | ~200-500ms   | Publish + PUBREC + PUBREL + PUBCOMP (2 RTT) |

**Network Conditions:** Unstable connection (packet loss 5%)

| QoS Level | Latency        | Explanation                            |
|-----------|----------------|----------------------------------------|
| QoS 0     | ~5-50ms        | No retries, message may be lost        |
| QoS 1     | ~200-2000ms    | Retries with exponential backoff       |
| QoS 2     | ~400-5000ms    | Multiple retries, 4-way handshake      |

---

### Battery Impact

**Scenario:** Remote lock command

| QoS Level | Network Usage | Battery Impact        |
|-----------|---------------|----------------------|
| QoS 0     | Minimal       | ~0.01 mAh            |
| QoS 1     | Medium        | ~0.02 mAh            |
| QoS 2     | High          | ~0.04 mAh            |

**Note:** For always-connected vehicles, battery impact is negligible. For vehicles that wake up for commands (SMS shoulder tap), the TCP/mTLS connection establishment (4.5 KB) dominates battery usage.

---

## Idempotency Requirements

### What is Idempotency?

An operation is **idempotent** if executing it multiple times has the same effect as executing it once.

**Idempotent Examples:**
- `SET door_lock = LOCKED` (same result regardless of repetitions)
- `GET location` (reading data is always safe)
- `DELETE session WHERE id = 123` (first delete succeeds, subsequent attempts have no effect)

**Non-Idempotent Examples:**
- `INCREMENT counter` (result changes with each execution)
- `TRANSFER $100 from account A to B` (repeating causes multiple transfers)
- `FLASH firmware` (repeating mid-operation can brick device)

---

### Implementing Idempotency

#### Method 1: Idempotency Keys (Recommended)

**Pattern:** Include a unique `idempotency_key` in every command request.

**Example:**
```protobuf
message RemoteCommandRequest {
    string idempotency_key = 1;  // UUID v4: "550e8400-e29b-41d4-a716-446655440000"
    string command = 2;           // "lock_doors"
    map<string, string> params = 3;
    int64 timestamp = 4;
}
```

**Vehicle-Side Implementation:**
```python
# Cache of processed idempotency keys (LRU cache, max 1000 entries, 24-hour TTL)
processed_keys = {}

def handle_command(request):
    key = request.idempotency_key

    # Check if already processed
    if key in processed_keys:
        # Return cached response (no-op)
        return processed_keys[key]

    # Process command
    response = execute_command(request.command, request.params)

    # Cache result
    processed_keys[key] = response
    return response
```

**Benefits:**
- Prevents duplicate execution even with QoS 1
- Allows safe retries from client side
- Enables caching of responses

---

#### Method 2: State Checks

**Pattern:** Check current state before executing command.

**Example: Lock Doors**
```python
def lock_doors(request):
    current_state = get_door_lock_state()

    if current_state == "LOCKED":
        # Already locked, return success (idempotent)
        return {"success": True, "message": "Doors already locked"}

    # Execute lock command
    actuate_door_locks("LOCK")
    return {"success": True, "message": "Doors locked"}
```

**Benefits:**
- Simple to implement
- No additional storage required
- Works well for stateful operations

**Limitations:**
- Does not prevent concurrent operations
- Race conditions possible

---

## Best Practices

### 1. Default to QoS 1

**Guideline:** Use QoS 1 for 80-90% of messages.

**Rationale:**
- Provides reliability without excessive overhead
- Handles network issues gracefully
- Acceptable latency for most use cases

---

### 2. Use QoS 2 Sparingly

**Guideline:** Only use QoS 2 for operations that are:
- Safety-critical (affects human life)
- Financially sensitive (billing, payments)
- Truly non-idempotent (cannot be safely repeated)

**Rationale:**
- 4x bandwidth overhead
- 2-5x latency increase
- Increased complexity (4-way handshake)

---

### 3. Implement Idempotency for QoS 1

**Guideline:** All QoS 1 commands MUST be idempotent.

**Implementation:**
- Use idempotency keys (UUIDs)
- Cache processed requests (LRU, 24-hour TTL)
- Return cached responses for duplicate requests

---

### 4. Set Appropriate Message Expiry

**Guideline:** Use MQTT 5.0 Message Expiry Interval to prevent stale commands.

**Examples:**
```python
# Real-time command: 30-second expiry
publish(
    topic="v2c/v1/us-east/VIN123/commands/lock-doors/request",
    qos=1,
    message_expiry_interval=30,  # seconds
    payload=command
)

# OTA notification: 7-day expiry
publish(
    topic="v2c/v1/us-east/VIN123/ota/available",
    qos=1,
    message_expiry_interval=604800,  # 7 days
    payload=ota_notification
)
```

---

### 5. Monitor Delivery Metrics

**Metrics to Track:**
- Message delivery success rate (per QoS level)
- Average latency (per QoS level)
- Retry count (QoS 1 and QoS 2)
- Duplicate message rate (QoS 1)
- Bandwidth consumption (per message type)

**Alerting Thresholds:**
- Delivery rate < 99% for QoS 1/2 → Alert
- Average latency > 2 seconds for QoS 1 → Investigate
- Duplicate rate > 5% for QoS 1 → Tune retry policy

---

### 6. Test Failure Scenarios

**Test Cases:**
- Network disconnection mid-transmission
- Broker restart during message delivery
- Client crash after receiving PUBLISH but before sending PUBACK
- Message expiry before delivery
- Duplicate message handling

---

## Testing and Validation

### Unit Tests

```python
def test_qos1_duplicate_handling():
    """Test that duplicate messages are handled idempotently"""
    command = RemoteCommandRequest(
        idempotency_key="test-key-123",
        command="lock_doors"
    )

    # First execution
    response1 = handle_command(command)
    assert response1.success == True
    assert get_door_state() == "LOCKED"

    # Duplicate execution (simulating QoS 1 retry)
    response2 = handle_command(command)
    assert response2 == response1  # Same response
    assert get_door_state() == "LOCKED"  # No state change
```

---

### Integration Tests

```python
def test_qos2_exactly_once():
    """Test that QoS 2 delivers exactly once"""
    broker = MQTTBroker()
    publisher = MQTTClient(qos=2)
    subscriber = MQTTClient(qos=2)

    received_messages = []
    subscriber.on_message = lambda msg: received_messages.append(msg)

    # Publish message
    publisher.publish("test/topic", "test-payload")

    # Wait for delivery
    time.sleep(1)

    # Verify exactly one delivery
    assert len(received_messages) == 1
```

---

### Chaos Testing

```python
def test_qos1_network_partition():
    """Test QoS 1 behavior during network partition"""
    vehicle = VehicleClient()
    cloud = CloudClient()

    # Vehicle publishes alert
    vehicle.publish(topic="v2c/v1/us-east/VIN123/events/battery-low", qos=1)

    # Simulate network partition
    network.disconnect(vehicle)

    # Wait for retry
    time.sleep(5)

    # Restore network
    network.connect(vehicle)

    # Verify message eventually delivered
    assert cloud.received_message(topic="v2c/v1/us-east/VIN123/events/battery-low")
```

---

## Examples

### Example 1: Remote Door Lock (QoS 1 with Idempotency)

**Client (Cloud):**
```python
import uuid
from mqtt_client import MQTTClient
from protobuf import RemoteCommandRequest

client = MQTTClient("cloud-service")
client.connect("broker.v2c.example.com", 8883)

# Generate idempotency key
idempotency_key = str(uuid.uuid4())

command = RemoteCommandRequest(
    idempotency_key=idempotency_key,
    command="lock_doors",
    timestamp=int(time.time() * 1000)
)

# Publish with QoS 1
client.publish(
    topic="v2c/v1/us-east/VIN12345/commands/lock-doors/request",
    qos=1,
    message_expiry_interval=30,  # 30 seconds
    payload=command.SerializeToString()
)
```

**Vehicle:**
```python
processed_keys = LRUCache(maxsize=1000, ttl=86400)  # 24 hours

def on_message(client, userdata, message):
    request = RemoteCommandRequest.FromString(message.payload)

    # Check idempotency
    if request.idempotency_key in processed_keys:
        # Return cached response
        return processed_keys[request.idempotency_key]

    # Execute command
    if request.command == "lock_doors":
        door_controller.lock()
        response = {"success": True, "timestamp": time.time()}
    else:
        response = {"success": False, "error": "Unknown command"}

    # Cache response
    processed_keys[request.idempotency_key] = response

    # Publish response
    client.publish(
        topic="v2c/v1/us-east/VIN12345/commands/lock-doors/response",
        qos=1,
        correlation_id=request.idempotency_key,
        payload=response
    )
```

---

### Example 2: OTA Install (QoS 2, Non-Idempotent)

**Cloud:**
```python
# OTA install is non-idempotent, use QoS 2
ota_install_request = OTAInstallRequest(
    update_id="firmware-v2.5.0",
    version="2.5.0",
    install_time=schedule_time  # Unix timestamp
)

client.publish(
    topic="v2c/v1/us-east/VIN12345/ota/install",
    qos=2,  # Exactly once
    message_expiry_interval=3600,  # 1 hour
    payload=ota_install_request.SerializeToString()
)
```

**Vehicle:**
```python
def on_ota_install_message(client, userdata, message):
    request = OTAInstallRequest.FromString(message.payload)

    # Check if already installed
    current_version = get_firmware_version()
    if current_version == request.version:
        # Already installed (QoS 2 prevents, but check anyway)
        return {"success": True, "message": "Already installed"}

    # Verify download integrity
    if not verify_firmware(request.update_id):
        return {"success": False, "error": "Integrity check failed"}

    # Perform installation (blocking, non-idempotent)
    install_firmware(request.update_id)

    # Verify installation
    new_version = get_firmware_version()
    if new_version == request.version:
        return {"success": True, "version": new_version}
    else:
        # Rollback on failure
        rollback_firmware()
        return {"success": False, "error": "Installation failed"}
```

---

### Example 3: High-Frequency Telemetry (QoS 0)

**Vehicle:**
```python
import time

while True:
    # Read GPS location
    location = gps.get_location()

    # Create telemetry message
    telemetry = VehiclePrecisionLocation(
        latitude=location.lat,
        longitude=location.lon,
        speed=vehicle.get_speed(),
        heading=vehicle.get_heading(),
        timestamp=int(time.time() * 1000)
    )

    # Publish with QoS 0 (fire-and-forget)
    client.publish(
        topic="v2c/v1/us-east/VIN12345/telemetry/location",
        qos=0,  # At most once
        payload=telemetry.SerializeToString()
    )

    # Sleep 60 seconds
    time.sleep(60)
```

---

## References

### Standards

- **MQTT 5.0 Specification:** https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- **MQTT QoS Explained:** https://www.hivemq.com/blog/mqtt-essentials-part-6-mqtt-quality-of-service-levels/
- **ISO 21434 - Cybersecurity:** https://www.iso.org/standard/70918.html

### Related Documentation

- [Topic Naming Convention](TOPIC_NAMING_CONVENTION.md)
- [Error Taxonomy (Common.proto)](../../src/main/proto/V2C/Common.proto)
- [FMEA: Vehicle Location Request](../fmea/FMEA_VEHICLE_LOCATION_REQUEST.md) - To be created

---

## Revision History

| Version | Date       | Author                     | Changes                          |
|---------|------------|----------------------------|----------------------------------|
| 1.0     | 2025-10-09 | V2C Architecture Team      | Initial draft                    |

---

## Approval

| Role                          | Name                | Signature | Date       |
|-------------------------------|---------------------|-----------|------------|
| Architecture Lead             | ___________________ | _________ | __________ |
| Vehicle Platform Lead         | ___________________ | _________ | __________ |
| Network Engineering Lead      | ___________________ | _________ | __________ |

---

**Document Classification:** Internal - Engineering Standards
**Security Level:** Confidential
**Distribution:** V2C Development Team, Vehicle Engineering, Cloud Platform Team
