# MQTT Topic Naming Convention Standard

**Version:** 1.0
**Status:** Draft
**Last Updated:** 2025-10-09
**Owner:** Vehicle Connectivity Architecture Team
**Standards Compliance:** ISO 21434, MQTT 5.0 Specification

---

## Table of Contents

1. [Overview](#overview)
2. [Topic Structure](#topic-structure)
3. [Component Definitions](#component-definitions)
4. [Message Direction Types](#message-direction-types)
5. [Topic Examples](#topic-examples)
6. [ACL Mapping](#acl-mapping)
7. [Validation Rules](#validation-rules)
8. [Best Practices](#best-practices)
9. [Performance Considerations](#performance-considerations)
10. [Migration Guide](#migration-guide)
11. [References](#references)

---

## Overview

This document defines the standard MQTT topic naming convention for the Vehicle-to-Cloud (V2C) Communications Architecture. A well-designed topic hierarchy is critical for:

- **Security:** Fine-grained access control via ACLs
- **Scalability:** Efficient routing and broker clustering
- **Observability:** Pattern-based monitoring and alerting
- **Maintainability:** Clear intent and easy debugging
- **Geographic Distribution:** Multi-region deployment support

### Design Principles

1. **Hierarchical:** Use `/` separators for tree-like structure enabling wildcard subscriptions
2. **ACL-Friendly:** Each level provides authorization granularity
3. **Scalable:** Support millions of vehicles across regions
4. **Versioned:** Allow protocol evolution without breaking existing clients
5. **Geographic:** Enable data residency and regional routing

---

## Topic Structure

### Standard Pattern

```
v2c/{version}/{region}/{vehicle_id}/{direction}/{message_type}[/{sub_type}]
```

### Structure Breakdown

| Position | Component      | Required | Max Length | Example              |
|----------|----------------|----------|------------|----------------------|
| 0        | Namespace      | Yes      | 10 chars   | `v2c`                |
| 1        | Version        | Yes      | 5 chars    | `v1`, `v2`           |
| 2        | Region         | Yes      | 20 chars   | `us-east`, `eu-west` |
| 3        | Vehicle ID     | Yes      | 64 chars   | `VIN12345ABC`        |
| 4        | Direction      | Yes      | 20 chars   | `telemetry`, `commands` |
| 5        | Message Type   | Yes      | 50 chars   | `location`, `diagnostics` |
| 6        | Sub Type       | Optional | 50 chars   | `request`, `response` |

**Maximum Topic Length:** 256 characters (MQTT 5.0 limit)

---

## Component Definitions

### 1. Namespace (`v2c`)

**Purpose:** Global namespace preventing collision with other MQTT systems on shared brokers.

**Valid Values:**
- `v2c` - Vehicle-to-Cloud communications

**Rationale:** Short (3 chars) to minimize bandwidth, distinctive to identify V2C traffic.

---

### 2. Version (`v1`, `v2`, etc.)

**Purpose:** Protocol versioning for backward compatibility during upgrades.

**Valid Values:**
- `v1` - Initial production version
- `v2`, `v3`, etc. - Future protocol iterations

**Migration Strategy:**
- Vehicles publish/subscribe to version-specific topics
- Cloud services support multiple concurrent versions during transition
- Deprecation policy: Minimum 24-month support for legacy versions

**Example Version Differences:**
- `v1`: Uses VehicleMessageHeader with correlation_id (int32)
- `v2`: Adds distributed tracing with trace_id (string, 128-bit)

---

### 3. Region (`us-east`, `eu-west`, etc.)

**Purpose:** Geographic routing for data residency compliance (GDPR, CCPA) and latency optimization.

**Valid Values:**
```
us-east    # US East Coast
us-west    # US West Coast
eu-west    # Europe West (Ireland, UK)
eu-central # Europe Central (Germany, France)
ap-south   # Asia Pacific South
ap-north   # Asia Pacific North
me-central # Middle East Central
sa-east    # South America East
```

**Routing Logic:**
- Vehicle publishes to region determined by GPS + provisioning
- Backend subscribes to specific regions or wildcard `v2c/v1/+/...`
- Regional brokers for data sovereignty

**Dynamic Region Switching:**
- Vehicle crossing regions: Disconnect, update topic prefix, reconnect
- Grace period: 30 minutes overlapping subscription during transition

---

### 4. Vehicle ID (`VIN12345ABC`)

**Purpose:** Unique vehicle identification for message routing and ACL enforcement.

**Format Options:**

#### Option A: Salted VIN Hash (Recommended)
```
SHA256(VIN + salt).substring(0, 16).toUpperCase()
Result: 3A7F2B9E4C1D6F8A
```

**Advantages:**
- PII compliance (VIN not transmitted in clear text)
- Fixed 16-character length
- Backend maintains VIN ↔ Hash mapping in secure database

#### Option B: Full VIN (ISO 3779 - 17 characters)
```
1HGBH41JXMN109186
```

**Use When:**
- VIN transmission approved by legal/privacy team
- Simplified debugging required (non-production)

**Constraints:**
- Must be globally unique
- Case-insensitive (converted to uppercase)
- No spaces or special characters
- Maximum 64 characters

---

### 5. Direction

**Purpose:** Categorize message flow direction for filtering and monitoring.

| Direction      | Description                                    | QoS Default |
|----------------|------------------------------------------------|-------------|
| `telemetry`    | Vehicle → Cloud (data upload, periodic)        | 0 or 1      |
| `commands`     | Cloud → Vehicle (remote control, immediate)    | 1 or 2      |
| `diagnostics`  | Vehicle → Cloud (on-demand, troubleshooting)   | 1           |
| `ota`          | Cloud ↔ Vehicle (firmware updates)             | 1 or 2      |
| `provisioning` | Cloud ↔ Vehicle (device onboarding)            | 2           |
| `events`       | Vehicle → Cloud (state changes, alerts)        | 1           |

---

### 6. Message Type

**Purpose:** Specific data or command category for routing and payload interpretation.

#### Telemetry Types

```
location         # GPS coordinates, speed, heading
battery          # State of charge, voltage, temperature
cabin            # Climate, seat position, window state
drivetrain       # Engine, transmission, braking
sensors          # LIDAR, camera, radar (ADAS)
connectivity     # Network signal, bandwidth usage
```

#### Command Types

```
lock-doors       # Remote lock/unlock
climate-control  # Pre-condition HVAC
horn-lights      # Activate horn/lights for locating vehicle
charge-control   # Start/stop charging (EV)
valet-mode       # Enable valet speed/distance limits
remote-start     # Engine start/stop
```

#### OTA Types

```
available        # Cloud notifies vehicle of available update
accept           # Vehicle confirms readiness to download
download         # Vehicle retrieves firmware package
install          # Vehicle begins installation
rollback         # Vehicle reverts to previous version
```

#### Diagnostics Types

```
dtc              # Diagnostic Trouble Codes
snapshot         # Full vehicle state capture
logs             # System logs upload
health           # Predictive maintenance signals
```

---

### 7. Sub Type (Optional)

**Purpose:** Request/response pattern or additional granularity.

**Common Values:**
- `request` - Initiate action (Cloud → Vehicle)
- `response` - Acknowledge/respond (Vehicle → Cloud)
- `status` - State update
- `error` - Error notification

---

## Message Direction Types

### Uplink (Vehicle → Cloud)

**Characteristics:**
- High volume (10-1000 messages/vehicle/day)
- QoS 0 for non-critical, QoS 1 for important
- Topic Aliases recommended (bandwidth savings)

**Topic Pattern:**
```
v2c/{version}/{region}/{vehicle_id}/telemetry/{message_type}
v2c/{version}/{region}/{vehicle_id}/events/{message_type}
v2c/{version}/{region}/{vehicle_id}/diagnostics/{message_type}
```

**Backend Subscription:**
```
v2c/v1/+/{vehicle_id}/telemetry/#    # Single vehicle, all telemetry
v2c/v1/us-east/+/telemetry/location  # All vehicles in region, location only
v2c/v1/+/+/events/#                  # All events, all regions
```

---

### Downlink (Cloud → Vehicle)

**Characteristics:**
- Low volume (0-10 messages/vehicle/day)
- QoS 1 minimum, QoS 2 for safety-critical
- Response Topic set for request/response pattern

**Topic Pattern:**
```
v2c/{version}/{region}/{vehicle_id}/commands/{message_type}/request
v2c/{version}/{region}/{vehicle_id}/ota/{message_type}
```

**Vehicle Subscription:**
```
v2c/v1/{region}/{vehicle_id}/commands/#
v2c/v1/{region}/{vehicle_id}/ota/#
```

---

### Bidirectional (Request/Response)

**Pattern:** MQTT 5.0 Response Topic

**Example Flow:**

**1. Cloud publishes command:**
```
Topic: v2c/v1/us-east/VIN12345/commands/lock-doors/request
Response Topic: v2c/v1/us-east/VIN12345/commands/lock-doors/response
Correlation ID: abc123-20251009T105530Z
Payload: { "lock": true }
```

**2. Vehicle subscribes to commands:**
```
Subscription: v2c/v1/us-east/VIN12345/commands/#
```

**3. Vehicle processes and responds:**
```
Topic: v2c/v1/us-east/VIN12345/commands/lock-doors/response
Correlation ID: abc123-20251009T105530Z
Payload: { "success": true, "timestamp": 1696853730 }
```

---

## Topic Examples

### Telemetry Examples

```
# GPS location update (periodic, every 60 seconds)
Topic: v2c/v1/us-east/3A7F2B9E4C1D6F8A/telemetry/location
QoS: 0
Payload: VehiclePrecisionLocation.proto

# Battery state change (event-driven, threshold crossed)
Topic: v2c/v1/eu-west/4B8G3C0F5D2E7G9B/events/battery
QoS: 1
Payload: BatteryStateChange.proto

# Diagnostic trouble code (alert)
Topic: v2c/v1/ap-south/5C9H4D1G6E3F8H0C/events/dtc
QoS: 1
Payload: DiagnosticTroubleCode.proto
```

---

### Command Examples

```
# Remote door lock command
Topic: v2c/v1/us-west/6D0J5E2H7F4G9J1D/commands/lock-doors/request
Response Topic: v2c/v1/us-west/6D0J5E2H7F4G9J1D/commands/lock-doors/response
QoS: 1
Payload: RemoteCommandRequest.proto

# Climate pre-conditioning
Topic: v2c/v1/eu-central/7E1K6F3J8G5H0K2E/commands/climate-control/request
Response Topic: v2c/v1/eu-central/7E1K6F3J8G5H0K2E/commands/climate-control/response
QoS: 1
Payload: ClimateControlRequest.proto
```

---

### OTA Update Examples

```
# Cloud notifies vehicle of available update
Topic: v2c/v1/us-east/8F2L7G4K9H6J1L3F/ota/available
QoS: 1
Payload: OTAUpdateAvailable.proto

# Vehicle acknowledges readiness
Topic: v2c/v1/us-east/8F2L7G4K9H6J1L3F/ota/accept
QoS: 1
Payload: OTAUpdateAccept.proto

# Vehicle reports installation progress
Topic: v2c/v1/us-east/8F2L7G4K9H6J1L3F/ota/install/status
QoS: 1
Payload: OTAInstallStatus.proto
```

---

## ACL Mapping

### Vehicle Client ACLs

**Principle:** Vehicles should have minimal privileges (least privilege principle).

#### Publish Permissions

```
v2c/v1/{region}/{vehicle_id}/telemetry/#    (Write)
v2c/v1/{region}/{vehicle_id}/events/#       (Write)
v2c/v1/{region}/{vehicle_id}/diagnostics/#  (Write)
v2c/v1/{region}/{vehicle_id}/commands/*/response (Write)
v2c/v1/{region}/{vehicle_id}/ota/accept     (Write)
v2c/v1/{region}/{vehicle_id}/ota/*/status   (Write)
```

#### Subscribe Permissions

```
v2c/v1/{region}/{vehicle_id}/commands/#     (Read)
v2c/v1/{region}/{vehicle_id}/ota/available  (Read)
v2c/v1/{region}/{vehicle_id}/ota/download   (Read)
```

**Key Security Properties:**
- Vehicle CANNOT subscribe to other vehicles' topics
- Vehicle CANNOT publish commands to itself (commands originate from backend)
- Vehicle CANNOT read other vehicles' responses

---

### Backend Service ACLs

#### Telemetry Ingestion Service

```
v2c/v1/+/+/telemetry/#                      (Read)
v2c/v1/+/+/events/#                         (Read)
```

#### Remote Command Service

```
v2c/v1/+/+/commands/*/request               (Write)
v2c/v1/+/+/commands/*/response              (Read)
```

#### OTA Service

```
v2c/v1/+/+/ota/available                    (Write)
v2c/v1/+/+/ota/download                     (Write)
v2c/v1/+/+/ota/accept                       (Read)
v2c/v1/+/+/ota/*/status                     (Read)
```

#### Monitoring Service (Read-Only)

```
v2c/#                                        (Read)
```

---

### ACL Implementation Examples

#### AWS IoT Core Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/us-east/${iot:ClientId}/telemetry/*",
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/us-east/${iot:ClientId}/events/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topicfilter/v2c/v1/us-east/${iot:ClientId}/commands/*"
      ]
    }
  ]
}
```

#### Eclipse Mosquitto ACL File

```
# Vehicle: 3A7F2B9E4C1D6F8A
user 3A7F2B9E4C1D6F8A
topic write v2c/v1/us-east/3A7F2B9E4C1D6F8A/telemetry/#
topic write v2c/v1/us-east/3A7F2B9E4C1D6F8A/events/#
topic read v2c/v1/us-east/3A7F2B9E4C1D6F8A/commands/#

# Backend: Telemetry Ingestion
user backend-telemetry-ingestion
topic read v2c/v1/+/+/telemetry/#
topic read v2c/v1/+/+/events/#
```

---

## Validation Rules

### Topic Validation Algorithm

```python
import re

TOPIC_PATTERN = re.compile(
    r'^v2c/'                           # Namespace
    r'v\d+/'                            # Version (v1, v2, etc.)
    r'[a-z]{2}-[a-z]+/'                 # Region (us-east, eu-west)
    r'[A-Z0-9]{8,64}/'                  # Vehicle ID (hash or VIN)
    r'(telemetry|commands|diagnostics|ota|events|provisioning)/' # Direction
    r'[a-z\-]+/'                        # Message type (kebab-case)
    r'([a-z\-]+)?$'                     # Optional sub-type
)

def validate_topic(topic: str) -> bool:
    if len(topic) > 256:
        return False
    if topic.count('/') < 5 or topic.count('/') > 6:
        return False
    return TOPIC_PATTERN.match(topic) is not None

# Valid
assert validate_topic("v2c/v1/us-east/VIN12345/telemetry/location")
assert validate_topic("v2c/v1/eu-west/HASH1234/commands/lock-doors/request")

# Invalid
assert not validate_topic("v2c/v1/us-east")                    # Too short
assert not validate_topic("v2c/v1/us-east/VIN/telemetry?")     # Special chars
assert not validate_topic("v2c/v1/US-EAST/VIN/telemetry/loc")  # Uppercase region
```

---

### Validation Requirements

| Component      | Validation Rule                                      |
|----------------|------------------------------------------------------|
| Namespace      | Must be exactly `v2c`                                |
| Version        | Must match `v\d+` (v1, v2, etc.)                     |
| Region         | Must match `[a-z]{2}-[a-z]+` (lowercase, hyphen)     |
| Vehicle ID     | Must match `[A-Z0-9]{8,64}` (uppercase alphanumeric) |
| Direction      | Must be in predefined list                           |
| Message Type   | Must match `[a-z\-]+` (lowercase, hyphen)            |
| Total Length   | Must be ≤ 256 characters                             |

---

## Best Practices

### 1. Use Topic Aliases (MQTT 5.0)

**Problem:** Repeated topic names waste bandwidth (35 bytes/message for long topics).

**Solution:** MQTT 5.0 Topic Aliases map topics to 2-byte integers.

**Bandwidth Savings:**
```
Without Alias:
  Topic: v2c/v1/us-east/3A7F2B9E4C1D6F8A/telemetry/location (52 bytes)
  Messages/day: 1440 (1/minute)
  Daily bandwidth: 52 * 1440 = 74,880 bytes = 73 KB

With Alias (after first message):
  Topic Alias: 0x0001 (2 bytes)
  Daily bandwidth: 52 + (2 * 1439) = 2,930 bytes = 2.9 KB

Savings: 96% bandwidth reduction
Fleet of 100,000 vehicles: 7.3 GB/day → 290 MB/day = 6.7 TB/year saved
```

**Implementation:**
```java
// First message: Establish alias
MqttPublish publish = new MqttPublish(
    "v2c/v1/us-east/VIN12345/telemetry/location",
    QoS.AT_MOST_ONCE,
    payload
);
publish.setTopicAlias(1);
client.publish(publish);

// Subsequent messages: Use alias
MqttPublish publishAlias = new MqttPublish(
    null,  // Empty topic
    QoS.AT_MOST_ONCE,
    payload
);
publishAlias.setTopicAlias(1);
client.publish(publishAlias);
```

---

### 2. Avoid Deep Topic Hierarchies

**Guideline:** Maximum 7 levels recommended (current: 6-7 levels).

**Rationale:**
- Broker memory overhead increases with depth
- Harder to debug and monitor
- Diminishing returns on ACL granularity

---

### 3. Use Wildcards for Efficient Subscriptions

**Single-Level Wildcard (`+`):**
```
v2c/v1/us-east/+/telemetry/location  # All vehicles in region
```

**Multi-Level Wildcard (`#`):**
```
v2c/v1/us-east/VIN12345/telemetry/#  # All telemetry for one vehicle
```

**Avoid:**
```
v2c/#  # Subscribes to ENTIRE V2C system (millions of messages)
```

---

### 4. Design for Monitoring

**Topic-Based Metrics:**
```
# Prometheus-style metrics from topic patterns
v2c_messages_total{direction="telemetry", type="location", region="us-east"}
v2c_command_latency_seconds{type="lock-doors", region="eu-west"}
```

**CloudWatch Logs Insights:**
```sql
fields @timestamp, topic
| filter topic like /v2c\/v1\/.*\/commands\/.*\/request/
| stats count() by bin(5m)
```

---

### 5. Plan for Multi-Region Failover

**Active-Active Regions:**
```
Primary:   v2c/v1/us-east/{vehicle_id}/...
Secondary: v2c/v1/us-west/{vehicle_id}/...
```

**Failover Logic:**
- Vehicle detects primary region unhealthy (connection timeout)
- Disconnects and updates topic prefix to secondary region
- Backend subscribes to multiple regions with `v2c/v1/+/{vehicle_id}/...`

---

## Performance Considerations

### Topic Matching Performance

**MQTT Broker Complexity:**
- Direct topic match: O(1)
- Wildcard match: O(n) where n = number of subscriptions

**Optimization:**
```
❌ Inefficient: 100,000 subscriptions to v2c/v1/+/+/telemetry/location
✅ Efficient: 1 shared subscription to v2c/v1/+/+/telemetry/location
```

---

### Storage Implications

**Topic in Retained Messages:**
- Topic stored with message payload
- Use short topics for retained messages (e.g., OTA update available)

**Example:**
```
Short topic:  v2c/v1/us/VIN/ota/avail      (27 bytes)
Long topic:   v2c/v1/us-east/VIN/ota/available (37 bytes)
Savings:      10 bytes × 100,000 vehicles = 1 MB
```

---

## Migration Guide

### Migrating from Unstructured Topics

**Current (Hypothetical):**
```
vehicle/{VIN}/data
vehicle/{VIN}/control
```

**Target V2C Structure:**
```
v2c/v1/us-east/{vehicle_id}/telemetry/{type}
v2c/v1/us-east/{vehicle_id}/commands/{type}/request
```

### Migration Strategy

**Phase 1: Dual Publishing (4 weeks)**
```
Vehicle publishes to BOTH old and new topics
Backend subscribes to BOTH
Monitor adoption metrics
```

**Phase 2: Monitoring (4 weeks)**
```
Alert if old topics still in use
Dashboard showing migration progress
```

**Phase 3: Deprecation Notice (8 weeks)**
```
Log warnings for vehicles using old topics
Email fleet operators
```

**Phase 4: Cutover (1 week)**
```
Remove old topic subscriptions
Vehicles auto-update to new topics (OTA)
```

---

## References

### Standards

- **MQTT 5.0 Specification:** https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
- **ISO 21434 - Cybersecurity Engineering:** https://www.iso.org/standard/70918.html
- **ISO 3779 - VIN Standard:** https://www.iso.org/standard/52200.html

### Industry Best Practices

- **AWS IoT Core Best Practices:** https://docs.aws.amazon.com/iot/latest/developerguide/security-best-practices.html
- **HiveMQ Topic Design:** https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/
- **Eclipse Mosquitto ACL Guide:** https://mosquitto.org/man/mosquitto-conf-5.html

### Related Documentation

- [Error Taxonomy (Common.proto)](../proto/V2C/Common.proto) - To be created
- [QoS Selection Guide](QOS_SELECTION_GUIDE.md) - To be created
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
| Security Architect            | ___________________ | _________ | __________ |
| ISO 21434 Compliance Officer  | ___________________ | _________ | __________ |

---

**Document Classification:** Internal - Engineering Standards
**Security Level:** Confidential
**Distribution:** V2C Development Team, Security Team, Operations Team
