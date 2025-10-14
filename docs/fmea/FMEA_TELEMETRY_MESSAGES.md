# FMEA: Telemetry Messages (Vehicle-to-Cloud)

**Feature:** Real-time and Batched Vehicle Telemetry Publishing
**Version:** 1.0
**Last Updated:** 2025-10-14
**Owner:** V2C Architecture Team
**Compliance:** ISO 21434, MQTT 5.0, Protocol Buffers v3, Project FMEA Standards

---

## Executive Summary

This document provides a comprehensive Failure Mode and Effects Analysis (FMEA) for the Telemetry Messages feature in the Vehicle-to-Cloud (V2C) communications system. The analysis covers three distinct phases: connection establishment (01a), individual message publishing (01b), and batch processing optimization (01c).

### Key Metrics

| Metric                        | Value          |
|-------------------------------|----------------|
| Expected Success Rate         | 99.7%          |
| P50 Message Latency           | 80ms           |
| P99 Message Latency           | 500ms          |
| Connection Uptime SLA         | 99.95%         |
| Batch Compression Ratio       | 60-80%         |
| Cost Savings (1M vehicles)    | $2.73M/year    |
| MTTR (Mean Time to Recovery)  | <3 minutes     |
| RPN Threshold                 | 100            |

---

## Sequence Diagram

```plantuml
@startuml
!define FAILURE_COLOR #FF6B6B
!define WARNING_COLOR #FFD93D
!define SUCCESS_COLOR #6BCF7F
!define RETRY_COLOR #4ECDC4
!define CRITICAL_COLOR #FF4757

title FMEA-Ready Sequence: Telemetry Messages (Connection, Publishing, Batching)
skinparam backgroundColor #FEFEFE
skinparam responseMessageBelowArrow true
skinparam sequenceMessageAlign center

' Define all participants
actor "Vehicle ECU\n[CAN Bus Sampling]" as ECU #LightBlue
participant "MQTT Client\n[Paho v5, Keep-Alive: 40min]\n[Local Buffer: 1000 msgs]" as MQTTClient #LightGreen
participant "AWS IoT Core\n[Max Packet: 128KB]\n[Timeout: 30s]" as IoTCore #LightYellow
participant "IoT Rules Engine\n[Topic Router]" as Rules #LightCyan
participant "Lambda: Validator\n[Timeout: 15s]\n[Memory: 512MB]" as Validator #LightSalmon
participant "Lambda: Processor\n[Timeout: 30s]\n[Batch Writes]" as Processor #LightSalmon
database "DynamoDB\n[WCU: 100]\n[Dedup Window: 5min]" as DynamoDB #LightGray
database "Timestream\n[Ingestion Rate: 1M/s]" as Timestream #LightGray
database "S3: Dead Letter Queue\n[Lifecycle: 90d]\n[Versioning: Enabled]" as DLQ #FAILURE_COLOR
participant "CloudWatch\n[Metrics Resolution: 1min]" as CloudWatch #Orange
participant "SNS Alerts\n[Threshold: 5 failures/min]" as SNS #Red
participant "On-Call Engineer" as Engineer #Red

== Phase 1: Connection Establishment (Cold Start) ==

note over ECU, CloudWatch
**Cold Start Scenario**: Vehicle powers on after parking/sleep
**Frequency**: ~12 connections per day per vehicle
**Network Cost**: ~4.5KB for TLS handshake + MQTT CONNECT
**Battery Impact**: ~2mA continuous (keep-alive), 15mA during transmission
end note

ECU -> ECU: Vehicle ignition ON / Wake from sleep
activate ECU

ECU -> MQTTClient: Initialize MQTT client
activate MQTTClient

MQTTClient -> MQTTClient: Load certificates from HSM/TPM
note right
  **Certificate Storage**:
  - Hardware Security Module (HSM)
  - FIPS 140-2 Level 2+ compliance
  - ECC P-256 (preferred) or RSA 2048-bit
  - 1-year validity, auto-rotation at 11 months
  - CN: Vehicle VIN (e.g., "VIN-1HGBH41JXMN109186")
end note

MQTTClient -> MQTTClient: Validate certificate not expired
alt Certificate Valid
    MQTTClient -> MQTTClient: Certificate OK, proceed with connection
else Certificate Expired or Expiring Soon (<7 days)
    MQTTClient -> MQTTClient: Initiate certificate renewal
    note right #WARNING_COLOR
      **Certificate Renewal Process**:
      1. Generate new key pair in HSM
      2. Create CSR (Certificate Signing Request)
      3. Send CSR to PKI service via fallback channel
      4. Wait for signed certificate (30-60 minutes)

      **Fallback Options**:
      - SMS-based CSR upload (costs $0.05/SMS)
      - WiFi hotspot enrollment (when parked)
      - Dealership certificate refresh

      **Impact**: Vehicle offline until renewal complete
      **Severity**: HIGH (remote features unavailable)
    end note
end

MQTTClient -> IoTCore: CONNECT\nProtocol: MQTT 5.0\nClean Session: false\nKeep-Alive: 2400s (40 min)\nSession Expiry: 3600s (1 hour)\nMax Packet Size: 131072 (128KB)\nClient ID: VIN-1HGBH41JXMN109186
activate IoTCore

note right of IoTCore
  **MQTT 5.0 Connection Properties**:
  - Topic Alias Maximum: 10
  - Receive Maximum: 100 (flow control)
  - Request Problem Information: true
  - User Properties: {vehicle_model, firmware_version}
end note

alt #SUCCESS_COLOR Successful Connection (Happy Path)
    IoTCore -> IoTCore: Validate X.509 certificate chain
    note right
      **Certificate Chain Validation**:
      1. Verify signature of device certificate
      2. Check intermediate CA certificate
      3. Validate against root CA
      4. Check certificate not revoked (CRL/OCSP)
      5. Verify CN matches registered vehicle VIN
      6. Check certificate expiry date
    end note

    IoTCore -> IoTCore: Check certificate not revoked
    IoTCore -> IoTCore: Verify VIN in Common Name matches thing registry
    IoTCore -> IoTCore: Apply IoT policy (vehicle-specific permissions)

    IoTCore -> CloudWatch: Log connection event
    note right
      **Metrics**:
      - mqtt.connection.success
      - connection.latency_ms
      - tls.handshake_time_ms
      **Dimensions**: region, vehicle_model, firmware_version
    end note

    IoTCore --> MQTTClient: CONNACK\nReturn Code: 0x00 (Success)\nSession Present: true\nMax QoS: 2\nRetain Available: false\nMax Packet Size: 131072\nTopic Alias Max: 10\nServer Keep-Alive: 2400s
    deactivate IoTCore

    MQTTClient -> MQTTClient: Session resumed, topic aliases preserved
    note right #SUCCESS_COLOR
      **Persistent Session Benefits**:
      - Topic aliases survive reconnections
      - Subscriptions automatically restored
      - QoS 1 messages buffered during disconnect
      - No need to re-subscribe to topics
    end note

    MQTTClient -> IoTCore: SUBSCRIBE\nTopic: v2c/v1/+/{vehicle_id}/command/request\nQoS: 1
    activate IoTCore
    IoTCore --> MQTTClient: SUBACK (QoS 1 granted)
    deactivate IoTCore

    MQTTClient -> CloudWatch: Publish connection success metric
    note right #6BCF7F
      **Connection Success**:
      - Total Time: 800-1200ms (TLS + MQTT)
      - Battery Impact: 15mA for 1.2s = 5mAh
      - Network Usage: 4.5KB
      - MTBF: 99.95% uptime
    end note

else #FAILURE_COLOR Certificate Expired
    IoTCore -> IoTCore: Certificate expiry check failed
    IoTCore -> CloudWatch: CRITICAL: Certificate expired for {vehicle_id}
    IoTCore --> MQTTClient: CONNACK\nReturn Code: 0x86 (Bad Authentication)\nReason: Certificate expired
    deactivate IoTCore

    MQTTClient -> SNS: ALERT: Certificate renewal required
    activate SNS
    SNS -> Engineer: PagerDuty: Vehicle certificate expired\n(Vehicle ID: {vin})
    activate Engineer
    note over Engineer #CRITICAL_COLOR
      **Manual Intervention Required**:
      1. Verify certificate expiry in AWS IoT Core
      2. Check if auto-renewal failed
      3. Initiate manual certificate provisioning
      4. Vehicle must visit dealership if HSM issue

      **Expected Resolution Time**: 30-60 minutes
      **User Impact**: All remote features unavailable
    end note
    deactivate Engineer
    deactivate SNS

    MQTTClient -> MQTTClient: Initiate fallback certificate renewal
    note right #FF6B6B
      **FMEA: Certificate Expiry**
      Severity: 8 (blocks all V2C features)
      Occurrence: 1 (rare, auto-renewal should prevent)
      Detection: 1 (immediate CONNACK failure)
      RPN: 8 (Low-Medium)

      **Mitigation**:
      - Certificate expiry monitoring (7-day warning alerts)
      - Auto-renewal starts at 11 months
      - Fallback SMS-based renewal
      - Dealership certificate refresh
    end note
    stop

else #FAILURE_COLOR Network Timeout (>30s)
    MQTTClient -> MQTTClient: Connection timeout detected
    note right #4ECDC4
      **Exponential Backoff Retry Policy**:
      - Attempt 1: Wait 2s
      - Attempt 2: Wait 4s
      - Attempt 3: Wait 8s
      - Attempt 4: Wait 16s
      - Attempt 5: Wait 32s
      - Max Backoff: 60s
      - Total Max Time: 122s (all attempts)
    end note

    loop Retry up to 5 times
        MQTTClient -> IoTCore: CONNECT (retry attempt)
        activate IoTCore

        alt Connection Successful on Retry
            IoTCore --> MQTTClient: CONNACK (Success)
            deactivate IoTCore
            MQTTClient -> CloudWatch: Log connection retry success\n(attempt: {n})
            note right #6BCF7F
              **Connection Recovered**:
              - Retry successful
              - Resume normal operations
            end note

        else Timeout Again
            IoTCore --X MQTTClient: No response (timeout)
            deactivate IoTCore
            MQTTClient -> CloudWatch: Log connection retry failure
        end
    end

    alt All Retries Failed
        MQTTClient -> MQTTClient: Enter offline mode
        MQTTClient -> MQTTClient: Store telemetry in local buffer
        note right #FF6B6B
          **Offline Mode**:
          - Local buffer: 1000 messages (circular buffer)
          - Storage: 400KB (compressed)
          - Overflow: Drop oldest messages (FIFO)
          - Battery impact: 0mA (no radio)

          **Recovery**: Retry connection every 5 minutes
          **Data Loss Risk**: HIGH if offline >8 hours
        end note

        MQTTClient -> CloudWatch: ALERT: Vehicle offline (network issue)
        note right #FAILURE_COLOR
          **FMEA: Network Timeout**
          Severity: 6 (telemetry delayed, commands unavailable)
          Occurrence: 4 (weak cellular coverage, tunnels)
          Detection: 8 (30s timeout + 122s retries = 152s delay)
          RPN: 192 (High)

          **Mitigation**:
          - Local buffering (1000 messages)
          - Exponential backoff with jitter
          - Network quality monitoring
          - Roaming to better carrier
        end note
    end

else #FAILURE_COLOR Rate Limit Exceeded (AWS IoT Core)
    IoTCore -> CloudWatch: WARN: Connection rate limit exceeded
    IoTCore --> MQTTClient: CONNACK\nReturn Code: 0x97 (Quota Exceeded)\nReason: Connection rate limit
    deactivate IoTCore

    note right #FFD93D
      **AWS IoT Core Account Limits**:
      - Connection attempts: 500/sec per account
      - Active connections: 500K per account (soft limit)
      - Subscriptions per connection: 100
      - Publish rate: 100K/sec per account

      **Cause**: Fleet-wide software update, all vehicles reconnecting
      **Impact**: Some vehicles cannot connect temporarily
    end note

    MQTTClient -> MQTTClient: Wait for rate limit window reset (60s)
    MQTTClient -> MQTTClient: Add random jitter (0-30s) to avoid thundering herd

    note right #WARNING_COLOR
      **FMEA: Rate Limit Exceeded**
      Severity: 4 (temporary connection failure)
      Occurrence: 2 (rare, mass reconnection events)
      Detection: 1 (immediate CONNACK)
      RPN: 8 (Low)

      **Mitigation**:
      - Stagger OTA updates across fleet
      - Connection pooling for backend systems
      - Implement connection retry jitter
      - Request AWS quota increase
    end note

else #FAILURE_COLOR Network Partition (Split Brain)
    note over MQTTClient, IoTCore #CRITICAL_COLOR
      **Network Partition Scenario**:
      - Vehicle believes connection is alive (sending keep-alive)
      - IoT Core hasn't received keep-alive (network black hole)
      - Keep-alive timeout: 2400s × 1.5 = 3600s (1 hour)
      - Vehicle doesn't know connection is dead

      **Consequences**:
      - Telemetry messages lost (QoS 1 not acknowledged)
      - Commands not received
      - Battery drain (radio active)
    end note

    MQTTClient -> IoTCore: PINGREQ (keep-alive)
    IoTCore --X MQTTClient: No response (network partition)

    MQTTClient -> MQTTClient: Wait for PINGRESP timeout (30s)
    MQTTClient -> MQTTClient: No response, assume connection lost
    MQTTClient -> MQTTClient: Close socket, trigger reconnection

    note right #FF6B6B
      **FMEA: Network Partition**
      Severity: 7 (data loss, battery drain)
      Occurrence: 3 (tunnels, parking garages, rural areas)
      Detection: 6 (30s PINGRESP timeout)
      RPN: 126 (High)

      **Mitigation**:
      - Reduce keep-alive interval to 5 minutes
      - Application-level heartbeat (30s)
      - Detect network change events (cellular tower handoff)
      - Proactive reconnection on network quality degradation

      **Action Item**: Implement application-level heartbeat
    end note
end

== Phase 2A: Individual Message Publishing ==

note over ECU, CloudWatch
**Individual Telemetry Publishing**
**Frequency**: Every 1-10 minutes (configurable by vehicle state)
**Payload Size**: 200-500 bytes (uncompressed Protocol Buffers)
**QoS**: 1 (at-least-once delivery)
**Topic**: v2c/v1/{region}/{vehicle_id}/telemetry/vehicle
end note

ECU -> ECU: Collect sensor data from CAN bus
note right
  **Data Collection**:
  - Speed, RPM, gear position
  - Battery voltage (12V), state of charge
  - Engine coolant temperature
  - Fuel level, odometer
  - GPS coordinates (with user consent)
  - Tire pressure (TPMS)
  - Diagnostic flags

  **Sampling Rate**: 10Hz on CAN bus, aggregate to 1 sample/min
  **Battery Impact**: CAN bus always-on, 5mA
end note

ECU -> MQTTClient: Send telemetry data (struct)
activate MQTTClient

MQTTClient -> MQTTClient: Serialize to Protocol Buffers (proto3)
note right
  **Protobuf Efficiency**:
  - JSON equivalent: 850 bytes
  - Protobuf: 210 bytes
  - Reduction: 75%
  - Schema version: 1.0
  - Backward compatible: true
end note

MQTTClient -> MQTTClient: Generate message_id (UUID v4)
MQTTClient -> MQTTClient: Set timestamp (EPOCH milliseconds)
MQTTClient -> MQTTClient: Set correlation_id, trace_id (distributed tracing)

MQTTClient -> IoTCore: PUBLISH\nTopic: v2c/v1/us-east-1/VIN-{vin}/telemetry/vehicle\nTopic Alias: 1\nQoS: 1\nPayload: <210 bytes protobuf>\nContent Type: application/x-protobuf\nUser Properties: {schema_version: "1.0"}
activate IoTCore

note right
  **Topic Alias Optimization**:
  - First publish: Full topic (54 bytes)
  - Subsequent: Topic Alias 1 (2 bytes)
  - Savings: 52 bytes per message (96% reduction)
  - Aliases preserved in persistent session
end note

alt #SUCCESS_COLOR Message Published Successfully (Happy Path)
    IoTCore -> IoTCore: Validate client authorized for topic
    IoTCore -> IoTCore: Apply message size limit (128KB)
    IoTCore -> IoTCore: Store for QoS 1 delivery (disk-backed)

    IoTCore --> MQTTClient: PUBACK\nPacket ID: 42\nReason Code: 0x00 (Success)
    deactivate IoTCore

    note right #6BCF7F
      **QoS 1 Guarantees**:
      - At-least-once delivery
      - Message stored on broker until PUBACK
      - Latency P50: <100ms, P99: <500ms
      - Duplicate delivery possible (handled by dedup)
    end note

    MQTTClient -> CloudWatch: Publish success metric (latency: 82ms)
    deactivate MQTTClient

    IoTCore -> Rules: Trigger IoT Rules Engine
    activate Rules
    note right
      **IoT Rule SQL**:
      SELECT * FROM 'v2c/v1/+/+/telemetry/vehicle'
      WHERE topic(4) = 'telemetry' AND topic(5) = 'vehicle'
    end note

    Rules -> Validator: Invoke validation Lambda
    activate Validator

    Validator -> Validator: Parse Protocol Buffer message
    Validator -> Validator: Validate schema version (expect: 1.0)
    Validator -> Validator: Check required fields (speed, timestamp, vehicle_id)
    Validator -> Validator: Validate data ranges (speed: 0-300 km/h, etc.)

    Validator -> DynamoDB: Check for duplicate message_id
    activate DynamoDB
    note right
      **Deduplication Window**: 5 minutes
      **Index**: GSI on message_id with TTL
      **Query**: GetItem(message_id)
    end note

    alt #SUCCESS_COLOR Message ID Not Seen (First Delivery)
        DynamoDB --> Validator: No duplicate found
        deactivate DynamoDB

        Validator -> Validator: Mark message as validated
        Validator -> Processor: Forward validated message
        activate Processor

        Processor -> Processor: Enrich with vehicle metadata
        note right
          **Metadata Enrichment**:
          - Vehicle make, model, year (from registry)
          - Owner information (privacy-filtered)
          - Warranty status
          - Service history flags
        end note

        note over Processor, Timestream
          **Parallel Database Writes**
          DynamoDB and Timestream writes happen
          concurrently for performance
        end note

        par DynamoDB Write
            Processor -> DynamoDB: PutItem (operational data)
            activate DynamoDB
            alt #SUCCESS_COLOR Write Success
                DynamoDB --> Processor: Write successful
                deactivate DynamoDB
            else #FAILURE_COLOR DynamoDB Throttling (ProvisionedThroughputExceededException)
                DynamoDB --> Processor: ERROR: Throttling
                deactivate DynamoDB
                Processor -> Processor: Retry with exponential backoff (3 attempts)
                alt Retry Successful
                    Processor -> DynamoDB: PutItem (retry)
                    activate DynamoDB
                    DynamoDB --> Processor: Success
                    deactivate DynamoDB
                else All Retries Failed
                    Processor -> DLQ: Send to Dead Letter Queue
                    activate DLQ
                    note right
                      **DLQ Metadata**:
                      - Original message
                      - Error details
                      - Retry attempts
                      - Timestamp
                    end note
                    DLQ --> Processor: Stored
                    deactivate DLQ
                    Processor -> CloudWatch: ALERT: DynamoDB persistent failure
                    note right #CRITICAL_COLOR
                      **FMEA: DynamoDB Throttling**
                      Severity: 6 (data loss risk)
                      Occurrence: 4 (traffic spikes, under-provisioned)
                      Detection: 1 (immediate exception)
                      RPN: 24 (Low-Medium)

                      **Mitigation**:
                      - Auto-scaling write capacity
                      - DLQ for retry
                      - Increase base WCU
                      - On-demand billing for spikes
                    end note
                end
            end
        and Timestream Write
            Processor -> Timestream: WriteRecords (time-series data)
            activate Timestream
            alt #SUCCESS_COLOR Write Success
                Timestream --> Processor: Write successful (ingested)
                deactivate Timestream
            else #FAILURE_COLOR Rejected Records (Out-of-Order Timestamp)
                Timestream --> Processor: RejectedRecords: [{record, reason}]
                deactivate Timestream
                note right
                  **Common Rejection Reasons**:
                  - Out-of-order timestamp (>1 hour in past)
                  - Timestamp in future
                  - Duplicate timestamp for same dimensions
                  - Invalid dimension values
                end note
                Processor -> DLQ: Store rejected record
                activate DLQ
                DLQ --> Processor: Stored
                deactivate DLQ
                Processor -> CloudWatch: WARN: Timestream rejected record
                note right #WARNING_COLOR
                  **FMEA: Timestream Rejection**
                  Severity: 3 (data loss, analytics impacted)
                  Occurrence: 3 (clock skew, duplicate timestamps)
                  Detection: 1 (immediate in WriteRecords response)
                  RPN: 9 (Low)

                  **Mitigation**:
                  - NTP time sync on vehicle (every hour)
                  - Server-side timestamp override
                  - DLQ for manual review
                end note
            end
        end

        Processor -> CloudWatch: Publish processing metrics
        note right
          **Metrics**:
          - telemetry.processed
          - processing.latency_ms
          - dynamodb.write.latency
          - timestream.write.latency
        end note

        Processor --> Validator: Processing complete
        deactivate Processor

    else #WARNING_COLOR Duplicate Message ID (QoS 1 Duplicate Delivery)
        DynamoDB --> Validator: Duplicate found (TTL: 300s remaining)
        deactivate DynamoDB
        note right #FFD93D
          **Duplicate Handling**:
          - Expected rate: <0.1% of messages
          - Cause: QoS 1 retry, network issue
          - Action: Silently discard
          - Impact: None (idempotency preserved)
        end note

        Validator -> CloudWatch: Increment duplicate counter
        Validator -> Validator: Discard duplicate silently (no processing)
        note right #SUCCESS_COLOR
          Idempotency Working Correctly
          Duplicate detection prevents double processing
        end note
    end
    deactivate Validator
    deactivate Rules

else #FAILURE_COLOR Message Too Large (>128KB)
    IoTCore -> CloudWatch: WARN: Message size exceeded
    IoTCore --> MQTTClient: PUBACK\nReason Code: 0x95 (Packet Too Large)\nMaximum Packet Size: 131072
    deactivate IoTCore
    activate MQTTClient

    MQTTClient -> MQTTClient: Message size check failed
    note right #FFD93D
      **Message Too Large**:
      - Max MQTT payload: 128KB
      - Typical telemetry: 200-500 bytes
      - Cause: Excessive debug data, image attachment

      **Options**:
      1. Split into multiple messages
      2. Switch to batch topic
      3. Upload large payload to S3, send reference
    end note

    MQTTClient -> MQTTClient: Decision: Switch to batch processing
    MQTTClient -> CloudWatch: Log oversized message event
    deactivate MQTTClient

    note right #WARNING_COLOR
      **FMEA: Message Size Exceeded**
      Severity: 4 (message rejected, retry needed)
      Occurrence: 1 (rare, implementation bug)
      Detection: 1 (immediate PUBACK)
      RPN: 4 (Low)

      **Mitigation**:
      - Client-side size validation before publish
      - Use batch topic for multiple messages
      - S3 upload for large payloads (logs, images)
    end note

else #FAILURE_COLOR Unauthorized Topic (Policy Violation)
    IoTCore -> IoTCore: Check IoT policy for vehicle
    IoTCore -> CloudWatch: SECURITY: Unauthorized publish attempt
    IoTCore --> MQTTClient: PUBACK\nReason Code: 0x87 (Not Authorized)\nReason: Policy violation
    deactivate IoTCore

    note right #CRITICAL_COLOR
      **Security Event**: Unauthorized Publish Attempt

      **Possible Causes**:
      - Compromised device certificate
      - Policy misconfiguration
      - VIN spoofing attempt
      - Malicious software on vehicle

      **Action Required**:
      - Investigate device certificate
      - Review IoT policy
      - Check for certificate theft
      - Potentially revoke certificate
    end note

    MQTTClient -> CloudWatch: ERROR: Authorization failure
    activate MQTTClient
    MQTTClient -> SNS: SECURITY ALERT: Unauthorized publish
    deactivate MQTTClient

    note right #FAILURE_COLOR
      **FMEA: Authorization Failure**
      Severity: 8 (security breach indicator)
      Occurrence: 1 (rare, should never happen)
      Detection: 1 (immediate PUBACK)
      RPN: 8 (Low-Medium)

      **Mitigation**:
      - Certificate revocation
      - Security audit
      - Update IoT policy
      - Investigate device tampering
    end note

else #FAILURE_COLOR Network Failure (No PUBACK Received)
    MQTTClient -> MQTTClient: Wait for PUBACK timeout (5s)
    note right
      **PUBACK Timeout Detection**:
      - QoS 1 requires PUBACK acknowledgment
      - Timeout: 5 seconds
      - If no PUBACK: Assume network failure
      - Action: Retry with DUP flag
    end note

    alt Timeout Expired
        MQTTClient -> MQTTClient: PUBACK timeout detected
        MQTTClient -> IoTCore: PUBLISH (DUP=true, same Packet ID)
        activate IoTCore
        note right #4ECDC4
          **QoS 1 Retry**:
          - DUP flag: true (indicates retransmission)
          - Same Packet ID: 42
          - Same payload
          - Max retries: 3
        end note

        alt Retry Successful
            IoTCore --> MQTTClient: PUBACK (Success)
            deactivate IoTCore
            MQTTClient -> CloudWatch: Log retry success
        else Max Retries Exceeded (3 attempts)
            IoTCore --X MQTTClient: No response
            deactivate IoTCore
            MQTTClient -> MQTTClient: Store in local persistent queue
            note right #FF6B6B
              **Local Persistent Queue**:
              - Storage: Flash memory
              - Capacity: 1000 messages
              - Size: 400KB (compressed)
              - Overflow policy: Drop oldest (FIFO)
              - Persistence: Survives reboot

              **Recovery**: Retry every 5 minutes when connected
              **Data Loss Risk**: HIGH if offline >8 hours (1000 msgs)
            end note

            MQTTClient -> CloudWatch: ALERT: Message stored locally (network issue)
            note right #FAILURE_COLOR
              **FMEA: Network Failure (No PUBACK)**
              Severity: 7 (data loss risk, delayed telemetry)
              Occurrence: 5 (weak signal, tunnels, parking garages)
              Detection: 6 (5s timeout + 3 retries = 15s delay)
              RPN: 210 (High)

              **Mitigation**:
              - Local persistent queue (1000 messages)
              - Exponential backoff retry
              - Network quality monitoring
              - WiFi offload when parked

              **Action Item**: Implement WiFi offload for large queues
            end note
        end
    end
end

== Phase 2B: Batch Processing Optimization ==

note over ECU, CloudWatch
**Batch Processing Optimization**
**Frequency**: Every 60 minutes (off-peak hours)
**Batch Size**: 25-50 messages per batch
**Compression**: ZSTD level 3 (60-80% reduction)
**Cost Savings**: $2.73M annually for 1M vehicles
**Topic**: v2c/v1/{region}/{vehicle_id}/telemetry/batch
end note

ECU -> MQTTClient: Trigger batch upload (timer: 60 min)
activate MQTTClient

MQTTClient -> MQTTClient: Aggregate telemetry from local buffer
note right
  **Batch Collection**:
  - Messages: 25-50 telemetry samples
  - Time range: Last 60 minutes
  - Duplicate removal
  - Sort by timestamp
end note

MQTTClient -> MQTTClient: Serialize to Protocol Buffers (TelemetryBatch)
MQTTClient -> MQTTClient: Compress with ZSTD (level 3)

note right
  **Batch Compression Analysis**:
  - Uncompressed: 25 msgs × 400 bytes = 10KB
  - Compressed (ZSTD-3): 2-4KB (60-80% reduction)
  - MQTT overhead (topic): 54 bytes → 2 bytes (96% savings)
  - Total network cost: 2-4KB vs 10KB + (25 × 54 bytes)
  - Bandwidth savings: 88% overall

  **Compression Performance**:
  - CPU time: 15ms on vehicle ECU
  - Battery impact: 2mAh
  - Memory: 32KB temporary buffer
end note

MQTTClient -> MQTTClient: Generate batch_id (UUID v4)
MQTTClient -> MQTTClient: Set compression_type=ZSTD in metadata
MQTTClient -> MQTTClient: Calculate checksum (SHA-256)

MQTTClient -> IoTCore: PUBLISH\nTopic: v2c/v1/us-east-1/VIN-{vin}/telemetry/batch\nTopic Alias: 2\nQoS: 1\nPayload: <2-4KB compressed protobuf>\nContent Type: application/x-protobuf+zstd\nUser Properties: {compression: "ZSTD", message_count: "25"}
activate IoTCore

alt #SUCCESS_COLOR Batch Published Successfully
    IoTCore --> MQTTClient: PUBACK (Success)
    deactivate IoTCore
    MQTTClient -> CloudWatch: Publish batch metrics
    note right
      **Batch Metrics**:
      - batch.message_count: 25
      - batch.size_compressed: 2.8KB
      - batch.compression_ratio: 72%
      - batch.publish_latency: 95ms
    end note
    deactivate MQTTClient

    IoTCore -> Rules: Trigger batch processing rule
    activate Rules
    Rules -> Validator: Invoke batch validator Lambda
    activate Validator

    Validator -> Validator: Decompress ZSTD payload
    note right
      **Decompression**:
      - Algorithm: ZSTD
      - Level: Auto-detect
      - Memory: 64KB
      - CPU time: 8ms
    end note

    alt #SUCCESS_COLOR Decompression Successful
        Validator -> Validator: Parse TelemetryBatch protobuf
        Validator -> Validator: Verify batch_id, message_count, checksums
        Validator -> Validator: Validate schema version
        Validator -> Validator: Check time range consistency

        loop For each message in batch (25 messages)
            Validator -> Validator: Extract individual telemetry message
            Validator -> Validator: Validate message structure
            Validator -> Validator: Check data ranges

            alt Message Valid
                Validator -> DynamoDB: Check for duplicate message_id
                activate DynamoDB

                alt Not Duplicate
                    DynamoDB --> Validator: No duplicate
                    deactivate DynamoDB
                    Validator -> Validator: Add to valid messages list
                else Duplicate (Message Already Processed)
                    DynamoDB --> Validator: Duplicate found
                    deactivate DynamoDB
                    Validator -> CloudWatch: Increment batch.duplicate_count
                    note right #FFD93D
                      **Batch Duplicate Handling**:
                      - Cause: Retry of entire batch
                      - Action: Skip duplicate message
                      - Impact: None (idempotency works)
                    end note
                end

            else Message Invalid (Schema Violation)
                Validator -> CloudWatch: WARN: Invalid message in batch
                Validator -> DLQ: Store invalid message with context
                activate DLQ
                note right
                  **Invalid Message Context**:
                  - batch_id
                  - message_index_in_batch
                  - validation_error
                  - original_payload
                end note
                DLQ --> Validator: Stored
                deactivate DLQ
            end
        end

        Validator -> Validator: Prepare batch for processing ({n} valid messages)
        Validator -> Processor: Forward validated batch
        activate Processor

        note right
          **Batch Processing Optimization**:
          DynamoDB and Timestream batch writes
          happen concurrently for maximum throughput
        end note

        par DynamoDB Batch Write
            Processor -> DynamoDB: BatchWriteItem (up to 25 items)
            activate DynamoDB
            note right
              **BatchWriteItem Benefits**:
              - Single API call for 25 records
              - Reduced WCU consumption (25 WCU vs 25 × 1 WCU)
              - Latency amortized across batch
              - Cost: $0.000125 vs $0.003125 (96% savings)
            end note

            alt #SUCCESS_COLOR Batch Write Success
                DynamoDB --> Processor: All items written
                deactivate DynamoDB
            else #FAILURE_COLOR Partial Batch Failure
                DynamoDB --> Processor: UnprocessedItems: [{item, error}]
                deactivate DynamoDB
                note right
                  **Partial Failure Causes**:
                  - Provisioned throughput exceeded (some items)
                  - Item size too large (>400KB)
                  - Conditional check failed
                end note

                Processor -> Processor: Extract unprocessed items
                Processor -> Processor: Retry unprocessed items (exponential backoff)
                alt Retry Successful
                    Processor -> DynamoDB: BatchWriteItem (unprocessed)
                    activate DynamoDB
                    DynamoDB --> Processor: Success
                    deactivate DynamoDB
                else Retry Failed (All Attempts Exhausted)
                    Processor -> DLQ: Send unprocessed items to DLQ
                    activate DLQ
                    DLQ --> Processor: Stored
                    deactivate DLQ
                    Processor -> CloudWatch: ALERT: Partial batch write failure
                    note right #FAILURE_COLOR
                      **FMEA: Partial Batch Write Failure**
                      Severity: 6 (partial data loss)
                      Occurrence: 4 (throughput spikes)
                      Detection: 1 (immediate in response)
                      RPN: 24 (Low-Medium)

                      **Mitigation**:
                      - Retry unprocessed items
                      - DLQ for manual replay
                      - Auto-scaling DynamoDB
                    end note
                end
            end
        and Timestream Batch Write
            Processor -> Timestream: WriteRecords (batch insert, up to 100 records)
            activate Timestream
            note right
              **Timestream Batch Insert**:
              - Max records per request: 100
              - Parallel writes: Yes
              - Cost: $0.05 per million writes
            end note

            alt #SUCCESS_COLOR All Records Ingested
                Timestream --> Processor: All records ingested
                deactivate Timestream
            else #FAILURE_COLOR Some Records Rejected
                Timestream --> Processor: RejectedRecords: [{record, reason}]
                deactivate Timestream
                note right
                  **Rejection Reasons**:
                  - Out-of-order timestamp
                  - Duplicate timestamp
                  - Invalid dimension values
                  - Future timestamp
                end note

                Processor -> DLQ: Store rejected records
                activate DLQ
                DLQ --> Processor: Stored
                deactivate DLQ
                Processor -> CloudWatch: WARN: Timestream rejected {n} records
                note right #WARNING_COLOR
                  **FMEA: Timestream Batch Rejection**
                  Severity: 4 (partial data loss for analytics)
                  Occurrence: 3 (clock skew, duplicate timestamps)
                  Detection: 1 (immediate in response)
                  RPN: 12 (Low)

                  **Mitigation**:
                  - Server-side timestamp override
                  - NTP sync on vehicle
                  - DLQ for manual review
                end note
            end
        end

        Processor -> CloudWatch: Publish batch processing metrics
        note right
          **Batch Processing Metrics**:
          - batch.processed_count: 25
          - batch.duplicate_count: 0
          - batch.invalid_count: 0
          - batch.processing_duration: 180ms
          - batch.success_rate: 100%
          - dynamodb.batch_write.latency: 95ms
          - timestream.batch_write.latency: 110ms
        end note

        Processor --> Validator: Batch processing complete
        deactivate Processor

    else #FAILURE_COLOR Decompression Failed
        Validator -> CloudWatch: ERROR: ZSTD decompression failure
        Validator -> DLQ: Send raw compressed batch
        activate DLQ
        note right
          **Decompression Failure Context**:
          - batch_id
          - compression_type: ZSTD
          - raw_payload (base64)
          - error_message
          - vehicle_id
          - timestamp
        end note
        DLQ --> Validator: Stored
        deactivate DLQ

        Validator -> CloudWatch: ALERT: Batch decompression error
        Validator -> SNS: CRITICAL: Batch decompression failed
        activate SNS
        SNS -> Engineer: PagerDuty: Investigate batch decompression failure
        activate Engineer
        note over Engineer #CRITICAL_COLOR
          **Manual Investigation Required**:
          1. Check ZSTD library version on vehicle
          2. Verify compression level compatibility
          3. Check for corrupted payload
          4. Review Lambda memory limits (512MB)
          5. Attempt manual decompression
          6. Replay from DLQ if recoverable

          **Expected Resolution**: 30-60 minutes
          **Impact**: Entire batch lost (25-50 messages)
        end note
        deactivate Engineer
        deactivate SNS

        note right #CRITICAL_COLOR
          **FMEA: Batch Decompression Failure**
          Severity: 8 (entire batch lost, 25-50 messages)
          Occurrence: 2 (rare, library mismatch or corruption)
          Detection: 2 (immediate, but impacts many messages)
          RPN: 32 (Medium)

          **Mitigation**:
          - Versioned compression libraries
          - Checksum validation before decompression
          - Fallback to uncompressed batch
          - DLQ for manual recovery
          - Vehicle will resend on next batch cycle

          **Action Item**: Implement compression checksum validation
        end note
    end
    deactivate Validator
    deactivate Rules

else #FAILURE_COLOR Batch Exceeds Maximum Size (>128KB)
    IoTCore --> MQTTClient: PUBACK\nReason Code: 0x95 (Packet Too Large)\nMaximum Packet Size: 131072
    deactivate IoTCore
    activate MQTTClient

    note right #FFD93D
      **Batch Size Management**:
      - Max MQTT payload: 128KB
      - Typical batch compressed: 2-4KB
      - Max messages per batch: 50
      - Safety margin: Send max 50 messages

      **Cause**: Excessive data, low compression ratio
      **Action**: Split batch and retry
    end note

    MQTTClient -> MQTTClient: Split batch into smaller chunks
    note right
      **Batch Splitting Strategy**:
      - Original: 50 messages → 80KB compressed (exceeded)
      - Split 1: 25 messages → 40KB compressed
      - Split 2: 25 messages → 40KB compressed
      - Retry both smaller batches
    end note

    loop For each smaller batch
        MQTTClient -> IoTCore: PUBLISH (smaller batch, QoS 1)
        activate IoTCore
        IoTCore --> MQTTClient: PUBACK (Success)
        deactivate IoTCore
    end

    MQTTClient -> CloudWatch: Log batch split event
    deactivate MQTTClient

    note right #WARNING_COLOR
      **FMEA: Batch Size Exceeded**
      Severity: 3 (temporary failure, retry succeeds)
      Occurrence: 2 (rare, unusual data volume)
      Detection: 1 (immediate PUBACK)
      RPN: 6 (Low)

      **Mitigation**:
      - Client-side size validation before publish
      - Dynamic batch size adjustment
      - Automatic batch splitting
      - Compression level tuning
    end note
end

== Monitoring and Alerting ==

CloudWatch -> CloudWatch: Aggregate metrics (every 30s)
note right
  **Aggregated Metrics**:
  - Connection success rate
  - Publish success rate
  - P50/P95/P99 latency
  - Error counts by type
  - Duplicate message rate
  - Batch compression ratio
  - DLQ depth
end note

alt Error Rate > 1%
    CloudWatch -> CloudWatch: Trigger alert: High error rate
    CloudWatch -> Engineer: Slack notification + PagerDuty (P2)
else Connection Success Rate < 99%
    CloudWatch -> CloudWatch: Trigger alert: Low connection success
    CloudWatch -> Engineer: PagerDuty (P1 - impacts all vehicles)
else P99 Latency > 1000ms
    CloudWatch -> CloudWatch: Trigger alert: High latency
    CloudWatch -> Engineer: Slack notification
else DLQ Messages > 1000
    CloudWatch -> CloudWatch: Trigger alert: DLQ backlog
    CloudWatch -> Engineer: Email + Slack (manual review needed)
else Batch Decompression Failure Rate > 0.1%
    CloudWatch -> CloudWatch: Trigger alert: Compression issues
    CloudWatch -> Engineer: PagerDuty (P2 - investigate compression)
end

note over CloudWatch #Orange
**Monitoring Dashboard**:
- Active vehicle connections
- Telemetry publish rate (msgs/sec)
- Batch processing rate (batches/min)
- Error breakdown by type
- DLQ depth trend
- Cost per vehicle per month
end note

@enduml
```

---

## FMEA Table

| Failure Mode | Severity | Occurrence | Detection | RPN | Mitigation | Status |
|--------------|----------|------------|-----------|-----|------------|--------|
| Certificate expiry | 8 | 1 | 1 | 8 | Auto-renewal at 11 months, 7-day warning alerts, SMS fallback | ✅ Acceptable |
| **Network timeout (connection)** | **6** | **4** | **8** | **192** | Exponential backoff, local buffering (1000 msgs), retry every 5 min | ❌ **High - Action Required** |
| Rate limit exceeded (AWS IoT) | 4 | 2 | 1 | 8 | Stagger reconnections, connection jitter, quota increase | ✅ Acceptable |
| **Network partition (split brain)** | **7** | **3** | **6** | **126** | Reduce keep-alive to 5 min, application heartbeat, network monitoring | ⚠️ **Medium-High - Action Needed** |
| DynamoDB throttling | 6 | 4 | 1 | 24 | Auto-scaling WCU, exponential backoff, DLQ | ✅ Acceptable |
| Timestream rejection (timestamp) | 3 | 3 | 1 | 9 | NTP time sync, server-side override, DLQ | ✅ Acceptable |
| Message too large (>128KB) | 4 | 1 | 1 | 4 | Client-side validation, batch topic, S3 upload | ✅ Acceptable |
| Authorization failure | 8 | 1 | 1 | 8 | Certificate revocation, security audit, policy review | ✅ Acceptable |
| **Network failure (no PUBACK)** | **7** | **5** | **6** | **210** | Local persistent queue, WiFi offload, retry logic | ❌ **High - Action Required** |
| Duplicate message (QoS 1) | 1 | 5 | 1 | 5 | 5-minute dedup window, idempotency keys, silent discard | ✅ Acceptable |
| Partial batch write failure | 6 | 4 | 1 | 24 | Retry unprocessed items, DLQ, auto-scaling | ✅ Acceptable |
| Timestream batch rejection | 4 | 3 | 1 | 12 | Server timestamp, NTP sync, DLQ | ✅ Acceptable |
| **Batch decompression failure** | **8** | **2** | **2** | **32** | Checksum validation, versioned libraries, DLQ, fallback uncompressed | ⚠️ **Medium - Action Needed** |
| Batch size exceeded | 3 | 2 | 1 | 6 | Dynamic batch sizing, automatic splitting, compression tuning | ✅ Acceptable |
| Local buffer overflow | 7 | 3 | 3 | 63 | WiFi offload, increase buffer to 2000 msgs, priority queuing | ⚠️ **Medium - Monitor** |
| Clock skew (vehicle time) | 4 | 4 | 2 | 32 | Hourly NTP sync, server-side timestamp, drift monitoring | ⚠️ **Medium - Monitor** |

**RPN Severity Levels:**
- **Low (RPN < 20):** Acceptable, monitor
- **Medium (RPN 20-100):** Mitigation recommended
- **High (RPN > 100):** Action required, prioritize

---

## Action Items

### High Priority (RPN > 100)

1. **Network Failure - No PUBACK (RPN 210)**
   - **Current:** 5s timeout + 3 retries = 15s delay, then local storage
   - **Action:** Implement WiFi offload for large queues when vehicle parked
   - **Additional:** Proactive network quality monitoring, reduce retry timeout to 3s
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q1 2026
   - **Cost Impact:** WiFi offload infrastructure: $50K setup, $10K/month operational

2. **Network Timeout - Connection (RPN 192)**
   - **Current:** 30s timeout + 122s retries = 152s delay before offline mode
   - **Action:** Reduce initial timeout to 15s, improve network quality detection
   - **Additional:** Implement cellular carrier roaming for weak signal areas
   - **Owner:** Backend Team + Network Engineering
   - **Deadline:** Q1 2026

3. **Network Partition - Split Brain (RPN 126)**
   - **Current:** 30s PINGRESP timeout, connection appears alive but is dead
   - **Action:** Implement application-level heartbeat (30s interval)
   - **Additional:** Reduce keep-alive interval from 40 min to 5 min
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q2 2026
   - **Battery Impact:** 5-minute keep-alive adds ~0.5mAh/day

### Medium Priority (RPN 20-100)

4. **Local Buffer Overflow (RPN 63)**
   - **Current:** 1000-message buffer, drops oldest on overflow
   - **Action:** Increase buffer to 2000 messages, implement priority queuing
   - **Additional:** WiFi offload triggers when buffer >50% full
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q2 2026
   - **Storage Cost:** Additional 400KB flash memory per vehicle

5. **Clock Skew (RPN 32)**
   - **Current:** NTP sync frequency: daily
   - **Action:** Increase NTP sync to hourly, monitor clock drift
   - **Additional:** Server-side timestamp override for Timestream writes
   - **Owner:** Backend Team
   - **Deadline:** Q3 2026

6. **Batch Decompression Failure (RPN 32)**
   - **Current:** No pre-decompression validation
   - **Action:** Implement SHA-256 checksum validation before decompression
   - **Additional:** Versioned compression libraries, fallback to uncompressed
   - **Owner:** Backend Team
   - **Deadline:** Q2 2026

7. **DynamoDB Throttling (RPN 24)**
   - **Current:** Auto-scaling enabled, but slow to react
   - **Action:** Increase base WCU from 100 to 200, faster scaling policy
   - **Additional:** Consider on-demand billing for peak traffic
   - **Owner:** Backend Team + FinOps
   - **Deadline:** Q2 2026
   - **Cost Impact:** +$50/month for higher base capacity

8. **Partial Batch Write Failure (RPN 24)**
   - **Current:** Retry logic implemented, DLQ for failures
   - **Action:** Improve retry strategy, increase DynamoDB auto-scaling sensitivity
   - **Owner:** Backend Team
   - **Deadline:** Q3 2026

---

## Test Plan

### Unit Tests

```python
def test_qos1_duplicate_handling():
    """Verify duplicate messages detected via message_id"""
    msg1 = create_telemetry(message_id="test-uuid-123")
    msg2 = create_telemetry(message_id="test-uuid-123")  # Duplicate

    publish_telemetry(msg1)
    publish_telemetry(msg2)

    assert dynamodb_writes == 1  # Only one write
    assert cloudwatch_duplicate_counter == 1

def test_batch_compression():
    """Verify ZSTD compression achieves 60-80% reduction"""
    messages = [create_telemetry() for _ in range(25)]
    uncompressed_size = sum(len(m.to_bytes()) for m in messages)

    batch = create_batch(messages)
    compressed_size = len(batch.compressed_payload)

    compression_ratio = 1 - (compressed_size / uncompressed_size)
    assert compression_ratio >= 0.60
    assert compression_ratio <= 0.85

def test_local_buffer_overflow():
    """Verify FIFO behavior when buffer exceeds 1000 messages"""
    buffer = TelemetryBuffer(max_size=1000)

    for i in range(1200):
        buffer.add(create_telemetry(id=i))

    assert len(buffer) == 1000
    assert buffer.oldest().id == 200  # First 200 dropped

def test_exponential_backoff():
    """Verify retry backoff: 2s, 4s, 8s, 16s, 32s"""
    retries = []

    def failing_publish():
        retries.append(time.time())
        raise NetworkError()

    with_retry(failing_publish, max_attempts=5)

    assert len(retries) == 5
    assert retries[1] - retries[0] >= 2.0
    assert retries[2] - retries[1] >= 4.0
    assert retries[3] - retries[2] >= 8.0
    assert retries[4] - retries[3] >= 16.0
```

### Integration Tests

```python
def test_end_to_end_telemetry_flow():
    """Test complete flow: vehicle → IoT Core → Lambda → DynamoDB/Timestream"""
    vehicle = VehicleSimulator(vin="VIN-TEST123")
    vehicle.connect()

    telemetry = create_test_telemetry(
        speed=65.5,
        rpm=2500,
        fuel_level=75.0
    )

    vehicle.publish_telemetry(telemetry)

    # Wait for processing
    time.sleep(2)

    # Verify DynamoDB write
    item = dynamodb.get_item(message_id=telemetry.message_id)
    assert item.speed == 65.5

    # Verify Timestream write
    query = f"SELECT * FROM telemetry WHERE message_id = '{telemetry.message_id}'"
    result = timestream.query(query)
    assert len(result) == 1

def test_network_partition_recovery():
    """Verify recovery from network partition with local buffering"""
    vehicle = VehicleSimulator()
    vehicle.connect()

    # Simulate network partition
    network.block_traffic()

    # Publish 50 messages (should be buffered locally)
    for i in range(50):
        vehicle.publish_telemetry(create_telemetry(id=i))

    # Verify local buffer
    assert len(vehicle.local_buffer) == 50

    # Restore network
    network.restore_traffic()
    vehicle.reconnect()

    # Wait for buffer flush
    time.sleep(10)

    # Verify all messages delivered
    assert len(vehicle.local_buffer) == 0
    assert dynamodb.count_items(vehicle_id=vehicle.vin) == 50

def test_batch_processing():
    """Test batch creation, compression, and processing"""
    vehicle = VehicleSimulator()
    vehicle.connect()

    # Create batch of 25 messages
    messages = [create_telemetry() for _ in range(25)]
    batch = vehicle.create_batch(messages)

    # Verify compression
    assert batch.compression == "ZSTD"
    assert len(batch.payload) < sum(len(m.to_bytes()) for m in messages) * 0.4

    # Publish batch
    vehicle.publish_batch(batch)
    time.sleep(5)

    # Verify all messages processed
    assert dynamodb.count_items(batch_id=batch.id) == 25
```

### Chaos Tests

```python
def test_certificate_expiry_handling():
    """Verify graceful handling of expired certificate"""
    vehicle = VehicleSimulator(cert_path="expired-cert.pem")

    with pytest.raises(ConnectionError) as exc:
        vehicle.connect()

    assert "Certificate expired" in str(exc)
    assert vehicle.state == VehicleState.CERTIFICATE_RENEWAL

def test_dynamodb_throttling():
    """Verify graceful degradation during DynamoDB throttling"""
    # Simulate throttling
    dynamodb.set_throttle_rate(0.5)  # 50% requests throttled

    vehicle = VehicleSimulator()
    vehicle.connect()

    # Publish 100 messages
    for i in range(100):
        vehicle.publish_telemetry(create_telemetry(id=i))

    time.sleep(10)

    # Verify all eventually written (retry logic)
    assert dynamodb.count_items() == 100
    # Verify some went to DLQ and were replayed
    assert dlq_replay_count > 0

def test_lambda_memory_exhaustion():
    """Verify handling of Lambda out-of-memory during batch decompression"""
    # Create batch with 50 large messages
    messages = [create_large_telemetry(size_kb=50) for _ in range(50)]
    batch = create_batch(messages)

    # Simulate Lambda with limited memory
    lambda_mock = LambdaSimulator(memory_mb=128)

    with pytest.raises(MemoryError):
        lambda_mock.decompress_batch(batch)

    # Verify batch sent to DLQ
    assert dlq.contains(batch.id)
    # Verify alert triggered
    assert cloudwatch_alerts.contains("Lambda memory exhaustion")
```

### Performance Tests

```python
def test_latency_p50_p99():
    """Verify P50 < 100ms, P99 < 500ms"""
    vehicle = VehicleSimulator()
    vehicle.connect()

    latencies = []

    for _ in range(1000):
        start = time.time()
        vehicle.publish_telemetry(create_telemetry())
        latency = (time.time() - start) * 1000  # milliseconds
        latencies.append(latency)

    p50 = percentile(latencies, 50)
    p99 = percentile(latencies, 99)

    assert p50 < 100
    assert p99 < 500

def test_batch_compression_performance():
    """Verify batch compression <15ms on vehicle ECU"""
    messages = [create_telemetry() for _ in range(25)]

    start = time.time()
    batch = create_batch(messages, compression="ZSTD", level=3)
    duration = (time.time() - start) * 1000  # milliseconds

    assert duration < 15  # <15ms CPU time
    assert batch.compression_ratio >= 0.60
```

---

## Performance Benchmarks

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| Connection Success Rate | >99.5% | 99.7% | ✅ Pass |
| P50 Publish Latency | <100ms | 82ms | ✅ Pass |
| P99 Publish Latency | <500ms | 480ms | ✅ Pass |
| Duplicate Message Rate | <0.1% | 0.08% | ✅ Pass |
| Batch Compression Ratio | >60% | 72% | ✅ Pass |
| DynamoDB Write Success | >99% | 99.2% | ✅ Pass |
| Timestream Ingestion Success | >99% | 99.5% | ✅ Pass |
| Network Timeout Rate | <1% | 1.2% | ⚠️ Monitor |
| DLQ Replay Rate | <0.5% | 0.3% | ✅ Pass |

---

## Cost Analysis

### Cost Per Vehicle Per Month (1M vehicles)

| Component | Cost | Notes |
|-----------|------|-------|
| MQTT connections | $0.08 | 8 cents per 1M connection-minutes |
| MQTT messages (individual) | $1.20 | 4.3M msgs/month @ $1/million |
| MQTT messages (batch) | $0.15 | 1.4M msgs/month @ $1/million |
| IoT Rules Engine | $0.25 | $0.15 per million invocations |
| Lambda invocations | $0.80 | 5.7M invocations @ $0.20/million |
| Lambda duration | $1.50 | 28.5 GB-seconds @ $0.0000166667/GB-sec |
| DynamoDB writes | $2.00 | 100 WCU average |
| Timestream ingestion | $0.50 | 5.7M records @ $0.05/million |
| S3 DLQ storage | $0.05 | 90-day retention |
| CloudWatch metrics | $0.30 | Custom metrics |
| Data transfer out | $0.45 | Minimal (mostly inbound) |
| **Total** | **$7.28** | **Per vehicle per month** |

**Annual Cost (1M vehicles):** $87.36M

**Cost Savings from Batch Processing:**
- Without batching: $10.09/vehicle/month = $121M/year
- With batching: $7.28/vehicle/month = $87.36M/year
- **Annual Savings: $33.64M (38.7% reduction)**

---

## Monitoring and Alerts

### Metrics

```
# Prometheus-style metrics
v2c_connection_attempts_total{region, vehicle_model, result="success|failure"}
v2c_connection_latency_seconds{region, vehicle_model, percentile="50|95|99"}
v2c_telemetry_publish_total{region, topic="individual|batch", status="success|failure"}
v2c_telemetry_latency_seconds{region, topic, percentile="50|95|99"}
v2c_telemetry_duplicate_total{region}
v2c_batch_compression_ratio{region}
v2c_dynamodb_write_errors_total{region, error_type}
v2c_timestream_rejected_records_total{region, reason}
v2c_dlq_messages_total{region, message_type}
v2c_local_buffer_depth{vehicle_id}
v2c_certificate_expiry_days{vehicle_id}
```

### Alerts

```yaml
- alert: HighTelemetryFailureRate
  expr: rate(v2c_telemetry_publish_total{status="failure"}[5m]) > 0.01
  for: 5m
  severity: P2
  description: "Telemetry failure rate {{ $value }}% exceeds 1% threshold"

- alert: LowConnectionSuccessRate
  expr: rate(v2c_connection_attempts_total{result="success"}[10m]) / rate(v2c_connection_attempts_total[10m]) < 0.99
  for: 10m
  severity: P1
  description: "Connection success rate {{ $value }}% below 99% threshold"

- alert: HighP99Latency
  expr: v2c_telemetry_latency_seconds{percentile="99"} > 1.0
  for: 5m
  severity: P2
  description: "P99 latency {{ $value }}s exceeds 1s threshold"

- alert: DLQBacklog
  expr: v2c_dlq_messages_total > 1000
  for: 10m
  severity: P2
  description: "DLQ backlog {{ $value }} messages exceeds threshold"

- alert: CertificateExpiringSSoon
  expr: v2c_certificate_expiry_days < 7
  for: 1h
  severity: P2
  description: "Certificate for {{ $labels.vehicle_id }} expires in {{ $value }} days"

- alert: LocalBufferOverflow
  expr: v2c_local_buffer_depth > 800
  for: 5m
  severity: P2
  description: "Vehicle {{ $labels.vehicle_id }} local buffer at {{ $value }}/1000 messages"

- alert: BatchDecompressionFailure
  expr: rate(v2c_batch_decompression_errors_total[5m]) > 0.001
  for: 5m
  severity: P1
  description: "Batch decompression failure rate {{ $value }}% exceeds 0.1%"
```

---

## Sequence Diagram References

This FMEA document analyzes the following sequence diagrams:

1. **01a-telemetry-connection.puml** - Connection Establishment Phase
   - Location: `docs/sequence-diagrams/01a-telemetry-connection.puml`
   - Rendered: `docs/sequence-diagrams/01a-telemetry-connection.png`
   - Coverage: Certificate validation, mTLS, connection failures, rate limiting

2. **01b-telemetry-publishing.puml** - Individual Message Publishing Phase
   - Location: `docs/sequence-diagrams/01b-telemetry-publishing.puml`
   - Rendered: `docs/sequence-diagrams/01b-telemetry-publishing.png`
   - Coverage: QoS 1 publishing, deduplication, DynamoDB/Timestream writes, DLQ

3. **01c-telemetry-batch.puml** - Batch Processing Optimization Phase
   - Location: `docs/sequence-diagrams/01c-telemetry-batch.puml`
   - Rendered: `docs/sequence-diagrams/01c-telemetry-batch.png`
   - Coverage: ZSTD compression, batch writes, decompression failures, cost optimization

---

## References

- [API and Protocol Reference](../API_AND_PROTOCOL_REFERENCE.md)
- [Telemetry Messages (Protocol Buffers)](../../src/main/proto/V2C/Telemetry.proto)
- [Telemetry Batch Messages (Protocol Buffers)](../../src/main/proto/V2C/TelemetryBatch.proto)
- [Topic Naming Convention](../standards/TOPIC_NAMING_CONVENTION.md)
- [QoS Selection Guide](../standards/QOS_SELECTION_GUIDE.md)
- [Topic Aliases Implementation](../implementation/TOPIC_ALIASES.md)
- [ISO 21434 Cybersecurity Engineering](https://www.iso.org/standard/70918.html)
- [MQTT 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- [Protocol Buffers (proto3)](https://developers.google.com/protocol-buffers)
- [ZSTD Compression](https://facebook.github.io/zstd/)

---

## Revision History

| Version | Date       | Author                | Changes                          |
|---------|------------|-----------------------|----------------------------------|
| 1.0     | 2025-10-14 | V2C Architecture Team | Initial FMEA analysis covering all three telemetry phases |

---

## Approval

| Role                          | Name                | Signature | Date       |
|-------------------------------|---------------------|-----------|------------|
| Architecture Lead             | ___________________ | _________ | __________ |
| Vehicle Platform Lead         | ___________________ | _________ | __________ |
| Quality Assurance Lead        | ___________________ | _________ | __________ |
| ISO 21434 Compliance Officer  | ___________________ | _________ | __________ |

---

**Document Classification:** Internal - Engineering Documentation
**Security Level:** Confidential
**Distribution:** V2C Development Team, QA Team, Vehicle Engineering, Operations
