# Vehicle-to-Cloud Communications API Reference

**Version:** 1.0.0
**Protocol:** MQTT 5.0 over TLS 1.3
**Serialization:** Protocol Buffers (proto3)
**Last Updated:** 2025-10-09

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Authentication](#authentication)
3. [Connection Setup](#connection-setup)
4. [Topic Structure](#topic-structure)
5. [Message Protocols](#message-protocols)
   - [Telemetry API](#telemetry-api)
   - [Commands API](#commands-api)
   - [OTA Updates API](#ota-updates-api)
   - [Diagnostics API](#diagnostics-api)
6. [Error Handling](#error-handling)
7. [Rate Limits](#rate-limits)
8. [Code Examples](#code-examples)
9. [Testing Guide](#testing-guide)
10. [Migration Guide](#migration-guide)

---

## Quick Start

### Prerequisites

- **MQTT Client**: Eclipse Paho, Mosquitto, or AWS IoT Device SDK
- **Protocol Buffers**: protoc compiler version 3.20+
- **TLS Certificates**: Device certificate, private key, Root CA
- **AWS IoT Core Endpoint**: `<account-id>.iot.<region>.amazonaws.com`

### 5-Minute Integration

```bash
# 1. Clone repository
git clone https://github.com/your-org/vehicle-to-cloud-architecture.git
cd vehicle-to-cloud-architecture

# 2. Generate protobuf code
protoc --java_out=src/main/java src/main/proto/V2C/*.proto

# 3. Install MQTT client
mvn install:install-file \
  -Dfile=org.eclipse.paho.mqttv5.client-1.2.5.jar \
  -DgroupId=org.eclipse.paho \
  -DartifactId=org.eclipse.paho.mqttv5.client \
  -Dversion=1.2.5 \
  -Dpackaging=jar

# 4. Configure connection
export AWS_IOT_ENDPOINT="a1b2c3d4e5f6g7.iot.us-east-1.amazonaws.com"
export VEHICLE_ID="VIN-1HGBH41JXMN109186"
export CERT_PATH="/path/to/device.crt"
export KEY_PATH="/path/to/device.key"
export CA_PATH="/path/to/AmazonRootCA1.pem"

# 5. Run sample application
java -jar v2c-client-sample.jar
```

---

## Authentication

### Mutual TLS (mTLS)

All connections MUST use mutual TLS with X.509 certificates.

**Certificate Requirements:**
- **Algorithm**: ECC P-256 or RSA 2048-bit
- **Validity**: 1 year (automatic rotation required)
- **Common Name (CN)**: Vehicle VIN
- **Key Usage**: Digital Signature, Key Encipherment
- **Extended Key Usage**: Client Authentication
- **Storage**: Hardware Security Module (HSM) or Trusted Platform Module (TPM)

**Certificate Chain:**
```
Root CA (offline, 20-year validity)
  └─ Intermediate CA (online, 10-year validity)
       └─ Device Issuing CA (5-year validity)
            └─ Vehicle Device Certificate (1-year validity, ECC P-256)
```

### Connection Policy

AWS IoT Core policy attached to device certificate:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:us-east-1:123456789012:client/${iot:Connection.Thing.ThingName}"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-${iot:Connection.Thing.ThingName}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topicfilter/v2c/v1/*/VIN-${iot:Connection.Thing.ThingName}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "iot:Receive",
      "Resource": [
        "arn:aws:iot:us-east-1:123456789012:topic/v2c/v1/*/VIN-${iot:Connection.Thing.ThingName}/*"
      ]
    }
  ]
}
```

---

## Connection Setup

### MQTT 5.0 Connection Parameters

```java
import org.eclipse.paho.mqttv5.client.*;
import org.eclipse.paho.mqttv5.common.MqttConnectOptions;

String broker = "ssl://a1b2c3d4e5f6g7.iot.us-east-1.amazonaws.com:8883";
String clientId = "VIN-1HGBH41JXMN109186";

MqttClient client = new MqttClient(broker, clientId);

MqttConnectionOptions options = new MqttConnectionOptions();
options.setMqttVersion(MqttConnectionOptions.MQTT_VERSION_5_0);
options.setCleanStart(false);  // Persistent session
options.setSessionExpiryInterval(3600L);  // 1 hour
options.setKeepAliveInterval(300);  // 5 minutes
options.setAutomaticReconnect(true);
options.setMaxReconnectDelay(60000);  // 60 seconds

// TLS configuration
options.setSocketFactory(createSSLSocketFactory(
    CERT_PATH,
    KEY_PATH,
    CA_PATH
));

// Topic alias configuration
MqttProperties connectProps = new MqttProperties();
connectProps.setTopicAliasMaximum(10);
options.setConnectionProperties(connectProps);

client.connect(options);
```

### Connection Lifecycle

```
┌─────────────┐
│   OFFLINE   │
└──────┬──────┘
       │ connect()
       ▼
┌─────────────┐    connectionLost()    ┌─────────────┐
│  CONNECTING ├───────────────────────►│ RECONNECTING│
└──────┬──────┘                        └──────┬──────┘
       │ connected()                          │
       ▼                                      │
┌─────────────┐◄───────────────────────────┐ │
│  CONNECTED  │        reconnected()         │ │
└──────┬──────┘                              │ │
       │ disconnect()                         │ │
       ▼                                      │ │
┌─────────────┐                              │ │
│ DISCONNECTING│                              │ │
└──────┬──────┘                              │ │
       │                                      │ │
       └──────────────────────────────────────┘ │
              │                                  │
              └──────────────────────────────────┘
```

### Connection Callbacks

```java
client.setCallback(new MqttCallback() {
    @Override
    public void connectionLost(Throwable cause) {
        log.error("Connection lost", cause);
        // Clear topic aliases (they're invalid now)
        topicAliasManager.reset();
        // Notify application layer
        notifyDisconnection();
    }

    @Override
    public void messageArrived(String topic, MqttMessage message) {
        handleIncomingMessage(topic, message.getPayload());
    }

    @Override
    public void deliveryComplete(IMqttToken token) {
        log.debug("Message delivered: {}", token.getMessageId());
    }

    @Override
    public void connectComplete(boolean reconnect, String serverURI) {
        if (reconnect) {
            log.info("Reconnected to {}", serverURI);
            resubscribeToTopics();
        } else {
            log.info("Connected to {}", serverURI);
            subscribeToTopics();
        }
    }
});
```

---

## Topic Structure

### Topic Naming Convention

```
v2c/v1/{region}/{vehicle_id}/{domain}/{message_type}
```

**Components:**
- `v2c`: Product identifier (Vehicle-to-Cloud)
- `v1`: API version
- `{region}`: AWS region (us-east-1, eu-west-1, ap-southeast-1)
- `{vehicle_id}`: VIN prefixed with "VIN-" (e.g., VIN-1HGBH41JXMN109186)
- `{domain}`: Message category (telemetry, command, ota, diagnostics)
- `{message_type}`: Specific message type

### Topic Hierarchy

```
v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/
├── telemetry/
│   ├── vehicle          (Vehicle → Cloud, QoS 1)
│   └── batch            (Vehicle → Cloud, QoS 1)
├── command/
│   ├── request          (Cloud → Vehicle, QoS 1-2)
│   ├── response         (Vehicle → Cloud, QoS 1)
│   ├── status           (Vehicle → Cloud, QoS 1)
│   └── cancel           (Cloud → Vehicle, QoS 2)
├── ota/
│   ├── available        (Cloud → Vehicle, QoS 2)
│   ├── accept           (Vehicle → Cloud, QoS 1)
│   ├── download         (Cloud → Vehicle, QoS 2)
│   ├── progress         (Vehicle → Cloud, QoS 1)
│   ├── install          (Cloud → Vehicle, QoS 2)
│   └── complete         (Vehicle → Cloud, QoS 1)
└── diagnostics/
    ├── request          (Cloud → Vehicle, QoS 1)
    ├── dtc              (Vehicle → Cloud, QoS 1)
    ├── snapshot         (Vehicle → Cloud, QoS 1)
    ├── health           (Vehicle → Cloud, QoS 1)
    └── logs             (Vehicle → Cloud, QoS 1)
```

### Subscription Patterns

```java
// Subscribe to all cloud→vehicle commands
String commandSubscription = "v2c/v1/+/VIN-1HGBH41JXMN109186/command/+";
client.subscribe(commandSubscription, 1);

// Subscribe to OTA updates
String otaSubscription = "v2c/v1/+/VIN-1HGBH41JXMN109186/ota/+";
client.subscribe(otaSubscription, 2);  // QoS 2 for safety-critical OTA

// Subscribe to diagnostic requests
String diagSubscription = "v2c/v1/+/VIN-1HGBH41JXMN109186/diagnostics/request";
client.subscribe(diagSubscription, 1);
```

---

## Message Protocols

### Telemetry API

#### Send Vehicle Telemetry

**Topic:** `v2c/v1/{region}/{vehicle_id}/telemetry/vehicle`
**Direction:** Vehicle → Cloud
**QoS:** 1 (at least once)
**Message Type:** `VehicleTelemetry`

**Example:**

```java
import com.vehicle.v2c.telemetry.v1.VehicleTelemetry;
import com.vehicle.v2c.common.v1.MessageMetadata;

// Build telemetry message
VehicleTelemetry telemetry = VehicleTelemetry.newBuilder()
    .setSpeed(65.5f)
    .setEngineRpm(2500)
    .setFuelLevel(75.0f)
    .setBatteryVoltage(12.6f)
    .setCoolantTemp(88.5f)
    .setOdometer(125430)
    .setGear("D")
    .setThrottlePosition(45.2f)
    .setMetadata(MessageMetadata.newBuilder()
        .setMessageId(UUID.randomUUID().toString())
        .setTimestamp(System.currentTimeMillis())
        .setVehicleId("VIN-1HGBH41JXMN109186")
        .setRegion("us-east-1")
        .build())
    .build();

// Serialize to protobuf
byte[] payload = telemetry.toByteArray();

// Publish with topic alias
String topic = "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/telemetry/vehicle";
MqttMessage message = new MqttMessage(payload);
message.setQos(1);

MqttProperties props = new MqttProperties();
props.setTopicAlias(1);  // Alias for telemetry topic

client.publish(topic, message, null, null, props);
```

**Frequency:** 1-10 messages per minute (configurable)
**Size:** 150-300 bytes (compressed)
**Retention:** 90 days (hot), 7 years (cold archive)

#### Send Batch Telemetry

**Topic:** `v2c/v1/{region}/{vehicle_id}/telemetry/batch`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `TelemetryBatch`

**Example:**

```java
import com.vehicle.v2c.telemetrybatch.v1.TelemetryBatch;

List<VehicleTelemetry> messages = new ArrayList<>();
// Collect 25 telemetry messages...

TelemetryBatch batch = TelemetryBatch.newBuilder()
    .setBatchId(UUID.randomUUID().toString())
    .setBatchTimestamp(System.currentTimeMillis())
    .setMessageCount(messages.size())
    .addAllTelemetryMessages(messages)
    .setCompression(CompressionType.ZSTD)
    .setTimeRange(TimeRange.newBuilder()
        .setStartTime(messages.get(0).getMetadata().getTimestamp())
        .setEndTime(messages.get(messages.size()-1).getMetadata().getTimestamp())
        .build())
    .build();

// Compress batch
byte[] compressed = compressZstd(batch.toByteArray());

client.publish(
    "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/telemetry/batch",
    compressed,
    1,
    false
);
```

**Batching Strategy:**
- Max 25-50 messages per batch
- Max 30 seconds between flushes
- Max 128KB batch size
- Use ZSTD compression (70% reduction)

---

### Commands API

#### Receive Remote Command

**Topic:** `v2c/v1/{region}/{vehicle_id}/command/request`
**Direction:** Cloud → Vehicle
**QoS:** 1-2 (depending on command criticality)
**Message Type:** `RemoteCommandRequest`

**Example:**

```java
client.subscribe("v2c/v1/+/VIN-1HGBH41JXMN109186/command/request", 2);

client.setCallback(new MqttCallback() {
    @Override
    public void messageArrived(String topic, MqttMessage message) {
        // Deserialize command
        RemoteCommandRequest request = RemoteCommandRequest.parseFrom(
            message.getPayload()
        );

        // Check for duplicate (idempotency)
        if (executedCommands.contains(request.getCommandId())) {
            log.warn("Duplicate command: {}", request.getCommandId());
            sendResponse(request.getCommandId(), ResponseStatus.DUPLICATE);
            return;
        }

        // Validate preconditions
        if (!validatePreconditions(request)) {
            sendResponse(request.getCommandId(), ResponseStatus.REJECTED);
            return;
        }

        // Execute command
        switch (request.getCommandCase()) {
            case LOCK_DOORS:
                executeLockDoors(request.getLockDoors());
                break;
            case UNLOCK_DOORS:
                executeUnlockDoors(request.getUnlockDoors());
                break;
            case START_CLIMATE:
                executeStartClimate(request.getStartClimate());
                break;
            // ... handle other commands
        }

        // Send acknowledgment
        sendResponse(request.getCommandId(), ResponseStatus.ACCEPTED);
    }
});
```

#### Send Command Response

**Topic:** `v2c/v1/{region}/{vehicle_id}/command/response`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `RemoteCommandResponse`

**Example:**

```java
void sendResponse(String commandId, ResponseStatus status) {
    RemoteCommandResponse response = RemoteCommandResponse.newBuilder()
        .setCommandId(commandId)
        .setStatus(status)
        .setVehicleState(VehicleCommandState.newBuilder()
            .setSpeedKmh(currentSpeed)
            .setIgnitionOn(ignitionState)
            .setDoorsOpen(areDoorsOpen())
            .setLocked(isLocked())
            .setBatteryLevel(batteryLevel)
            .build())
        .setMetadata(MessageMetadata.newBuilder()
            .setMessageId(UUID.randomUUID().toString())
            .setTimestamp(System.currentTimeMillis())
            .build())
        .build();

    client.publish(
        "v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/command/response",
        response.toByteArray(),
        1,
        false
    );
}
```

#### Common Commands

##### Lock Doors

```java
RemoteCommandRequest lockRequest = RemoteCommandRequest.newBuilder()
    .setCommandId(UUID.randomUUID().toString())
    .setUserId("user-12345")
    .setLockDoors(LockDoorsCommand.newBuilder()
        .setLockTrunk(true)
        .setEnableAlarm(true)
        .setFlashLights(true)
        .setHonkHorn(false)
        .build())
    .setPriority(CommandPriority.NORMAL)
    .setTimeoutSeconds(30)
    .build();

cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/command/request",
             lockRequest, 1);
```

##### Start Climate

```java
RemoteCommandRequest climateRequest = RemoteCommandRequest.newBuilder()
    .setCommandId(UUID.randomUUID().toString())
    .setUserId("user-12345")
    .setStartClimate(StartClimateCommand.newBuilder()
        .setTargetTemperatureCelsius(22.0f)
        .setMode(ClimateMode.AUTO)
        .setFanSpeed(0)  // Auto
        .setSeatHeaterEnabled(true)
        .setSeatHeaterLevel(2)
        .setAutoShutoffMinutes(15)
        .build())
    .setPriority(CommandPriority.NORMAL)
    .setTimeoutSeconds(60)
    .build();

cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/command/request",
             climateRequest, 1);
```

##### Remote Start (Critical - QoS 2)

```java
RemoteCommandRequest startRequest = RemoteCommandRequest.newBuilder()
    .setCommandId(UUID.randomUUID().toString())
    .setUserId("user-12345")
    .setRemoteStart(RemoteStartCommand.newBuilder()
        .setClimateSettings(StartClimateCommand.newBuilder()
            .setTargetTemperatureCelsius(21.0f)
            .build())
        .setAutoShutoffMinutes(10)
        .setBatteryPreconditioningEnabled(true)
        .setPinCode("1234")  // Additional security
        .build())
    .setPriority(CommandPriority.HIGH)
    .setTimeoutSeconds(120)
    .setAuthorizationToken("jwt-token-here")  // Required for critical commands
    .build();

// QoS 2 for safety-critical remote start
cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/command/request",
             startRequest, 2);
```

---

### OTA Updates API

#### Notify Update Available

**Topic:** `v2c/v1/{region}/{vehicle_id}/ota/available`
**Direction:** Cloud → Vehicle
**QoS:** 2 (exactly once, safety-critical)
**Message Type:** `OTAUpdateAvailable`

**Example:**

```java
OTAUpdateAvailable update = OTAUpdateAvailable.newBuilder()
    .setUpdateId(UUID.randomUUID().toString())
    .setVersion("2.5.0")
    .setUpdateType(UpdateType.SECURITY_PATCH)
    .setPriority(UpdatePriority.HIGH)
    .setSizeBytes(52428800)  // 50 MB
    .setReleaseNotes("Critical security update. Fixes CVE-2025-12345.")
    .setMinBatteryLevel(30)
    .setCanInstallWhileDriving(false)
    .setEstimatedInstallTimeSec(600)  // 10 minutes
    .setExpiresAt(System.currentTimeMillis() + 7 * 24 * 3600 * 1000L)  // 7 days
    .build();

cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/ota/available",
             update, 2);
```

#### Accept/Reject Update

**Topic:** `v2c/v1/{region}/{vehicle_id}/ota/accept`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `OTAUpdateAccept`

**Example:**

```java
// User accepts update
OTAUpdateAccept accept = OTAUpdateAccept.newBuilder()
    .setUpdateId(update.getUpdateId())
    .setStatus(AcceptanceStatus.ACCEPTED)
    .setScheduledInstallTime(System.currentTimeMillis() + 3600000)  // 1 hour
    .setVehicleState(VehicleState.newBuilder()
        .setPowerState(PowerState.ACCESSORY)
        .setIsMoving(false)
        .setIgnitionOn(false)
        .setParkingBrakeEngaged(true)
        .build())
    .setBatteryLevel(85)
    .setAvailableStorageBytes(5368709120L)  // 5 GB
    .build();

vehiclePublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/ota/accept",
               accept, 1);

// User defers update
OTAUpdateAccept defer = OTAUpdateAccept.newBuilder()
    .setUpdateId(update.getUpdateId())
    .setStatus(AcceptanceStatus.DEFERRED)
    .setReason("User postponed to tomorrow")
    .setScheduledInstallTime(System.currentTimeMillis() + 86400000)  // 24 hours
    .build();
```

#### Download Update

**Topic:** `v2c/v1/{region}/{vehicle_id}/ota/download`
**Direction:** Cloud → Vehicle
**QoS:** 2
**Message Type:** `OTADownloadRequest`

**Example:**

```java
// Generate presigned S3 URL (valid for 1 hour)
String downloadUrl = s3Client.generatePresignedUrl(
    "v2c-ota-updates",
    "firmware/v2.5.0/update.bin",
    Date.from(Instant.now().plus(1, ChronoUnit.HOURS))
);

OTADownloadRequest download = OTADownloadRequest.newBuilder()
    .setUpdateId(updateId)
    .setDownloadUrl(downloadUrl)
    .setChecksumSha256("a3f5b9c1e2d4...")
    .setSignature(ByteString.copyFrom(hsmSignature))
    .setSizeBytes(52428800)
    .setSupportsResume(true)
    .setChunkSizeBytes(1048576)  // 1 MB chunks
    .setMaxDownloadTimeSec(3600)  // 1 hour
    .build();

cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/ota/download",
             download, 2);
```

#### Report Download Progress

**Topic:** `v2c/v1/{region}/{vehicle_id}/ota/progress`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `OTADownloadProgress`

**Example:**

```java
// Send progress every 10%
OTADownloadProgress progress = OTADownloadProgress.newBuilder()
    .setUpdateId(updateId)
    .setStatus(DownloadStatus.IN_PROGRESS)
    .setBytesDownloaded(26214400)  // 25 MB
    .setTotalBytes(52428800)  // 50 MB
    .setSpeedBps(1048576)  // 1 MB/s
    .setEtaSeconds(25)
    .build();

vehiclePublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/ota/progress",
               progress, 1);
```

---

### Diagnostics API

#### Request Diagnostic Session

**Topic:** `v2c/v1/{region}/{vehicle_id}/diagnostics/request`
**Direction:** Cloud → Vehicle
**QoS:** 1
**Message Type:** `DiagnosticSessionRequest`

**Example:**

```java
DiagnosticSessionRequest request = DiagnosticSessionRequest.newBuilder()
    .setSessionId(UUID.randomUUID().toString())
    .setSessionType(SessionType.COMPREHENSIVE)
    .addAllRequestedDiagnostics(Arrays.asList(
        DiagnosticType.DTCS,
        DiagnosticType.ECU_HEALTH,
        DiagnosticType.BATTERY_HEALTH,
        DiagnosticType.TPMS
    ))
    .setMaxDurationSeconds(300)  // 5 minutes
    .setPriority(SessionPriority.NORMAL)
    .build();

cloudPublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/diagnostics/request",
             request, 1);
```

#### Report DTCs

**Topic:** `v2c/v1/{region}/{vehicle_id}/diagnostics/dtc`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `DTCReport`

**Example:**

```java
DTCReport report = DTCReport.newBuilder()
    .setReportId(UUID.randomUUID().toString())
    .setTimestamp(System.currentTimeMillis())
    .setReportType(DTCReportType.ACTIVE_DTCS)
    .addDtcs(DiagnosticTroubleCode.newBuilder()
        .setCode("P0171")
        .setDescription("System Too Lean (Bank 1)")
        .setSeverity(DTCSeverity.MEDIUM)
        .setCategory(DTCCategory.POWERTRAIN)
        .setStatus(DTCStatus.ACTIVE)
        .setFirstOccurrence(System.currentTimeMillis() - 86400000)  // 1 day ago
        .setOccurrenceCount(3)
        .setFreezeFrame(FreezeFrame.newBuilder()
            .setEngineRpm(2200)
            .setSpeedKmh(80.0f)
            .setCoolantTempCelsius(92.5f)
            .setThrottlePositionPercent(42.0f)
            .build())
        .setAffectedSystem("Fuel System")
        .setRecommendedAction("Check for vacuum leaks, inspect MAF sensor")
        .build())
    .setTotalDtcCount(1)
    .setSourceEcu("ECM")
    .setMileageKm(125430)
    .build();

vehiclePublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/diagnostics/dtc",
               report, 1);
```

#### Send Vehicle Snapshot

**Topic:** `v2c/v1/{region}/{vehicle_id}/diagnostics/snapshot`
**Direction:** Vehicle → Cloud
**QoS:** 1
**Message Type:** `VehicleSnapshot`

**Example:**

```java
VehicleSnapshot snapshot = VehicleSnapshot.newBuilder()
    .setSnapshotId(UUID.randomUUID().toString())
    .setTimestamp(System.currentTimeMillis())
    .setReason(SnapshotReason.PERIODIC)
    .setTrigger("Daily health check")
    .setTelemetry(getCurrentTelemetry())
    .addAllActiveDtcs(getActiveDTCs())
    .addAllEcuHealth(getAllECUHealth())
    .setBatteryHealth(BatteryHealth.newBuilder()
        .setVoltage12v(12.6f)
        .setSoc12vPercent(95.0f)
        .setSoh12vPercent(98.0f)
        .setTemperature12vCelsius(25.0f)
        .build())
    .setTpms(TirePressureMonitoring.newBuilder()
        .setFrontLeft(TireData.newBuilder()
            .setPressureKpa(220.0f)
            .setTemperatureCelsius(32.0f)
            .setStatus(TireStatus.NORMAL)
            .build())
        // ... other tires
        .build())
    .build();

vehiclePublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/diagnostics/snapshot",
               snapshot, 1);
```

---

## Error Handling

### V2CError Structure

All error responses use standardized `V2CError` message from Common.proto:

```protobuf
message V2CError {
  ErrorCode code = 1;
  string message = 2;
  string details = 3;
  repeated string stack_trace = 4;
  map<string, string> metadata = 5;
}
```

### Error Codes

| Code | Name | HTTP Equiv | Description |
|------|------|------------|-------------|
| 0 | UNKNOWN | 500 | Unknown error |
| 1 | INVALID_REQUEST | 400 | Malformed request |
| 2 | UNAUTHORIZED | 401 | Authentication failed |
| 3 | FORBIDDEN | 403 | Insufficient permissions |
| 4 | NOT_FOUND | 404 | Resource not found |
| 5 | CONFLICT | 409 | Resource conflict |
| 6 | PRECONDITION_FAILED | 412 | Precondition not met |
| 7 | RATE_LIMIT_EXCEEDED | 429 | Too many requests |
| 8 | INTERNAL_ERROR | 500 | Internal server error |
| 9 | SERVICE_UNAVAILABLE | 503 | Service unavailable |
| 10 | TIMEOUT | 504 | Request timeout |

### Error Handling Example

```java
try {
    RemoteCommandRequest request = RemoteCommandRequest.parseFrom(payload);

    // Validate preconditions
    if (currentSpeed > 5.0f && request.hasUnlockDoors()) {
        V2CError error = V2CError.newBuilder()
            .setCode(ErrorCode.PRECONDITION_FAILED)
            .setMessage("Cannot unlock doors while moving")
            .setDetails("Current speed: " + currentSpeed + " km/h")
            .putMetadata("max_speed_allowed", "5.0")
            .putMetadata("current_speed", String.valueOf(currentSpeed))
            .build();

        sendErrorResponse(request.getCommandId(), error);
        return;
    }

    // Execute command
    executeCommand(request);

} catch (InvalidProtocolBufferException e) {
    V2CError error = V2CError.newBuilder()
        .setCode(ErrorCode.INVALID_REQUEST)
        .setMessage("Failed to parse command request")
        .setDetails(e.getMessage())
        .build();

    sendErrorResponse(null, error);
}
```

### Retry Strategy

```java
class RetryPolicy {
    private static final int MAX_RETRIES = 3;
    private static final int BASE_DELAY_MS = 1000;

    public void executeWithRetry(Callable<Void> operation) {
        int attempt = 0;

        while (attempt < MAX_RETRIES) {
            try {
                operation.call();
                return;  // Success

            } catch (TransientException e) {
                attempt++;
                if (attempt >= MAX_RETRIES) {
                    throw new RuntimeException("Max retries exceeded", e);
                }

                // Exponential backoff: 1s, 2s, 4s
                int delay = BASE_DELAY_MS * (1 << (attempt - 1));
                log.warn("Retry attempt {} after {}ms", attempt, delay);
                Thread.sleep(delay);

            } catch (PermanentException e) {
                // Don't retry permanent errors
                throw new RuntimeException("Permanent error", e);
            }
        }
    }
}
```

---

## Rate Limits

### Vehicle → Cloud Limits

| Message Type | Limit | Window | Burst |
|-------------|-------|--------|-------|
| Telemetry (individual) | 10/min | 1 min | 20 |
| Telemetry (batch) | 2/min | 1 min | 5 |
| Command Response | 100/min | 1 min | 150 |
| DTC Reports | 10/hour | 1 hour | 20 |
| Snapshots | 1/hour | 1 hour | 3 |

### Cloud → Vehicle Limits

| Message Type | Limit | Window | Burst |
|-------------|-------|--------|-------|
| Commands | 10/min | 1 min | 15 |
| OTA Requests | 1/day | 24 hours | 1 |
| Diagnostic Requests | 5/hour | 1 hour | 10 |

### Rate Limit Response

```java
// When rate limit exceeded
RateLimitExceeded response = RateLimitExceeded.newBuilder()
    .setResetAt(System.currentTimeMillis() + 60000)  // Resets in 1 minute
    .setRetryAfterSeconds(60)
    .setRateLimitInfo(RateLimitInfo.newBuilder()
        .setCommandsSent(10)
        .setMaxCommands(10)
        .setWindowSeconds(60)
        .setResetInSeconds(45)
        .build())
    .build();

// Publish rate limit error
vehiclePublish("v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/command/response",
               response, 1);
```

---

## Code Examples

### Complete Vehicle Client (Java)

```java
public class V2CVehicleClient {
    private final MqttClient mqttClient;
    private final TopicAliasManager aliasManager;
    private final String vehicleId;
    private final String region;

    public V2CVehicleClient(String endpoint, String vehicleId, String region,
                            String certPath, String keyPath, String caPath)
            throws Exception {
        this.vehicleId = vehicleId;
        this.region = region;

        // Initialize MQTT client
        String broker = "ssl://" + endpoint + ":8883";
        this.mqttClient = new MqttClient(broker, vehicleId);

        // Configure connection
        MqttConnectionOptions options = new MqttConnectionOptions();
        options.setMqttVersion(MqttConnectionOptions.MQTT_VERSION_5_0);
        options.setCleanStart(false);
        options.setSessionExpiryInterval(3600L);
        options.setKeepAliveInterval(300);
        options.setAutomaticReconnect(true);
        options.setSocketFactory(SSLUtil.createSocketFactory(certPath, keyPath, caPath));

        MqttProperties connectProps = new MqttProperties();
        connectProps.setTopicAliasMaximum(10);
        options.setConnectionProperties(connectProps);

        // Set callback
        mqttClient.setCallback(new V2CCallback());

        // Connect
        mqttClient.connect(options);

        // Initialize topic alias manager
        this.aliasManager = new TopicAliasManager(region, vehicleId);

        // Subscribe to cloud→vehicle topics
        subscribeToTopics();
    }

    private void subscribeToTopics() throws Exception {
        String baseTopic = "v2c/v1/+/" + vehicleId;

        mqttClient.subscribe(baseTopic + "/command/request", 2);
        mqttClient.subscribe(baseTopic + "/command/cancel", 2);
        mqttClient.subscribe(baseTopic + "/ota/#", 2);
        mqttClient.subscribe(baseTopic + "/diagnostics/request", 1);
    }

    public void sendTelemetry(VehicleTelemetry telemetry) throws Exception {
        String topic = String.format("v2c/v1/%s/%s/telemetry/vehicle",
                                    region, vehicleId);

        MqttPublishMessage msg = aliasManager.preparePublish(
            topic, telemetry.toByteArray(), 1);

        MqttMessage mqttMsg = new MqttMessage(msg.payload);
        mqttMsg.setQos(msg.qos);

        MqttProperties props = new MqttProperties();
        if (msg.topicAlias != null) {
            props.setTopicAlias(msg.topicAlias);
        }

        mqttClient.publish(msg.topic, mqttMsg, null, null, props);
    }

    private class V2CCallback implements MqttCallback {
        @Override
        public void messageArrived(String topic, MqttMessage message) {
            try {
                if (topic.contains("/command/request")) {
                    handleCommandRequest(
                        RemoteCommandRequest.parseFrom(message.getPayload())
                    );
                } else if (topic.contains("/ota/available")) {
                    handleOTAAvailable(
                        OTAUpdateAvailable.parseFrom(message.getPayload())
                    );
                } else if (topic.contains("/diagnostics/request")) {
                    handleDiagnosticRequest(
                        DiagnosticSessionRequest.parseFrom(message.getPayload())
                    );
                }
            } catch (Exception e) {
                log.error("Failed to process message", e);
            }
        }

        @Override
        public void connectionLost(Throwable cause) {
            log.error("Connection lost", cause);
            aliasManager.reset();
        }

        @Override
        public void deliveryComplete(IMqttToken token) {
            log.debug("Message delivered: {}", token.getMessageId());
        }

        @Override
        public void connectComplete(boolean reconnect, String serverURI) {
            log.info("Connected to {}", serverURI);
            if (reconnect) {
                try {
                    subscribeToTopics();
                } catch (Exception e) {
                    log.error("Failed to resubscribe", e);
                }
            }
        }
    }
}
```

---

## Testing Guide

### Unit Testing

```java
@Test
public void testTelemetryPublish() throws Exception {
    // Mock MQTT client
    MqttClient mockClient = mock(MqttClient.class);

    V2CVehicleClient client = new V2CVehicleClient(mockClient);

    VehicleTelemetry telemetry = VehicleTelemetry.newBuilder()
        .setSpeed(60.0f)
        .setEngineRpm(2000)
        .build();

    client.sendTelemetry(telemetry);

    // Verify publish was called
    verify(mockClient).publish(
        eq("v2c/v1/us-east-1/VIN-TEST/telemetry/vehicle"),
        any(MqttMessage.class),
        isNull(),
        isNull(),
        any(MqttProperties.class)
    );
}
```

### Integration Testing

```java
@Test
public void testCommandExecution() throws Exception {
    // Connect real client to test broker
    V2CVehicleClient client = new V2CVehicleClient(
        TEST_ENDPOINT,
        "VIN-TEST",
        "us-east-1",
        TEST_CERT,
        TEST_KEY,
        TEST_CA
    );

    // Send lock command from cloud simulator
    RemoteCommandRequest lockCommand = RemoteCommandRequest.newBuilder()
        .setCommandId(UUID.randomUUID().toString())
        .setLockDoors(LockDoorsCommand.newBuilder()
            .setLockTrunk(true)
            .build())
        .build();

    cloudSimulator.publish(
        "v2c/v1/us-east-1/VIN-TEST/command/request",
        lockCommand.toByteArray(),
        1
    );

    // Wait for response
    RemoteCommandResponse response = cloudSimulator.waitForResponse(5000);

    assertEquals(ResponseStatus.COMPLETED, response.getStatus());
}
```

---

## Migration Guide

### From MQTT 3.1.1 to MQTT 5.0

**Changes Required:**

1. **Update Client Library**
```xml
<!-- Old -->
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>

<!-- New -->
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.mqttv5.client</artifactId>
    <version>1.2.5</version>
</dependency>
```

2. **Update Connection Code**
```java
// Old
MqttConnectOptions options = new MqttConnectOptions();
options.setMqttVersion(MqttConnectOptions.MQTT_VERSION_3_1_1);

// New
MqttConnectionOptions options = new MqttConnectionOptions();
options.setMqttVersion(MqttConnectionOptions.MQTT_VERSION_5_0);
MqttProperties props = new MqttProperties();
props.setTopicAliasMaximum(10);
options.setConnectionProperties(props);
```

3. **Add Topic Aliases**
```java
// New feature in MQTT 5.0
TopicAliasManager aliasManager = new TopicAliasManager(region, vehicleId);
```

**Benefits:**
- 96% bandwidth savings with topic aliases
- Better error reporting
- Session expiry control
- Reason codes for all failures

---

**Last Updated:** 2025-10-09
**API Version:** 1.0.0
**Support:** api-support@example.com
