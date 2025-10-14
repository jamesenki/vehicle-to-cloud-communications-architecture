# FMEA: OTA (Over-The-Air) Updates

**Feature:** OTA Software and Firmware Updates for Connected Vehicles
**Version:** 1.0
**Last Updated:** 2025-10-14
**Owner:** V2C Architecture Team
**Compliance:** ISO 21434, MQTT 5.0, A/B Partition Architecture, Project FMEA Standards

---

## Executive Summary

This document provides a comprehensive Failure Mode and Effects Analysis (FMEA) for the Over-The-Air (OTA) update system in the Vehicle-to-Cloud (V2C) communications architecture. The analysis covers the complete end-to-end flow across three phases: notification and acceptance (03a), secure download with resume capability (03b), and installation with automatic rollback protection (03c).

### Key Metrics

| Metric                        | Value          |
|-------------------------------|----------------|
| Expected Success Rate         | 98.0%          |
| P95 Download Time (50MB)      | 14 minutes     |
| P99 Installation Time         | 12 minutes     |
| Maximum Total Update Time     | 60 minutes     |
| MTTR (Mean Time to Recovery)  | <10 minutes    |
| RPN Threshold                 | 100            |
| Rollback Success Rate         | 99.9%          |

---

## Sequence Diagram

```plantuml
@startuml
!define FAILURE_COLOR #FF6B6B
!define WARNING_COLOR #FFD93D
!define SUCCESS_COLOR #6BCF7F
!define RETRY_COLOR #4ECDC4
!define CRITICAL_COLOR #FF4757

title FMEA-Ready Sequence: OTA Update Complete Flow (Notification → Download → Installation)
skinparam backgroundColor #FEFEFE
skinparam responseMessageBelowArrow true
skinparam sequenceMessageAlign center

' Define all participants
actor "Fleet Manager" as Manager #LightBlue
participant "OTA Service\n[Campaign Orchestration]" as OTAService #LightGreen
database "DynamoDB\n[Campaign Tracking]\n[Connection Pool: 50]" as DynamoDB #LightGray
participant "S3\n[Update Repository]\n[Multi-Region]" as S3 #LightYellow
participant "HSM Signing\n[AWS CloudHSM]\n[ECC P-256]" as HSM #LightPink
queue "AWS IoT Core\n[MQTT 5.0]\n[QoS: 0,1,2]" as IoTCore #LightCyan
participant "MQTT Client\n[Vehicle ECU]\n[Persistent Session]" as MQTT #LightBlue
participant "Update Agent\n[A/B Partitions]\n[Timeout: 3600s]" as UpdateAgent #LightGreen
participant "Vehicle HSM/TPM\n[Signature Verification]" as VehicleHSM #LightPink
database "Vehicle Storage\n[Flash Memory]\n[Partition A/B]" as Storage #LightGray
participant "Diagnostics\n[Health Checks]" as Diagnostics #LightSalmon
queue "DLQ\n[Failed Updates]\n[Retention: 14d]" as DLQ #FAILURE_COLOR
participant "Monitoring\n[CloudWatch/Datadog]\n[Alert Threshold: 5 failures/100]" as Monitor #Orange
participant "On-Call Engineer" as Engineer #Red

== Phase 1: OTA Campaign Creation and Notification ==

Manager -> OTAService: Create OTA Campaign
activate OTAService
note right
{update_id: "ota-2025-001",
version: "2.5.1",
type: SECURITY_PATCH,
priority: CRITICAL,
target_filter: "model='Accord' AND year>=2023",
rollout_strategy: CANARY,
canary_size: 100,
min_battery_level: 30,
estimated_install_time: 600}
end note

OTAService -> OTAService: Validate campaign parameters\n(version format, target filter syntax)

alt Validation Success
    OTAService -> S3: Verify update package exists
    activate S3

    alt Package Found
        S3 --> OTAService: 200 OK\n{size: 52428800 bytes (50MB), etag: "abc123"}
        deactivate S3

        OTAService -> HSM: Sign update metadata\n(SHA-256 hash + ECC P-256 signature)
        activate HSM

        alt HSM Available
            HSM --> OTAService: Digital Signature (256 bytes)
            deactivate HSM

            OTAService -> DynamoDB: BEGIN TRANSACTION
            activate DynamoDB

            OTAService -> DynamoDB: INSERT INTO campaigns\n(id, version, status='CREATED', created_at)
            DB --> OTAService: 1 row inserted

            OTAService -> DynamoDB: Query vehicles matching filter\n(model='Accord' AND year>=2023)
            DynamoDB --> OTAService: 5000 vehicles matched

            OTAService -> DynamoDB: COMMIT
            deactivate DynamoDB

            OTAService -> OTAService: Select canary group\n(First 100 vehicles, stratified by region)

            OTAService -> DynamoDB: Create update_status records\n(100 records, status='PENDING')
            activate DynamoDB
            DynamoDB --> OTAService: Records created
            deactivate DynamoDB

        else HSM Timeout (>5s)
            HSM --X OTAService: Timeout (no response)
            deactivate HSM
            OTAService -> OTAService: Retry HSM signing\n(Attempt 2 of 3, exponential backoff)

            alt Retry Successful
                OTAService -> HSM: Sign update metadata (retry)
                activate HSM
                HSM --> OTAService: Digital Signature
                deactivate HSM
            else All Retries Failed
                OTAService -> Monitor: CRITICAL: HSM signing service down
                activate Monitor
                Monitor -> Monitor: Alert threshold exceeded\n(P1 incident)
                Monitor -> Engineer: PagerDuty: HSM unavailable
                activate Engineer
                note over Engineer #CRITICAL_COLOR
                  Manual Intervention Required:
                  1. Check AWS CloudHSM cluster status
                  2. Verify HSM network connectivity
                  3. Check HSM key material
                  4. Failover to secondary HSM if available
                  5. Campaign creation blocked until resolved
                end note
                deactivate Engineer
                deactivate Monitor

                OTAService --> Manager: 503 Service Unavailable\n{error: "HSM signing service unavailable"}
                deactivate OTAService
                note over Manager, HSM #CRITICAL_COLOR
                  FMEA: HSM Signing Failure
                  Severity: 9 (blocks ALL OTA updates)
                  Occurrence: 1 (rare, AWS outage)
                  Detection: 1 (immediate)
                  RPN: 9 (Low-Medium)
                  Mitigation: Multi-region HSM, health checks, auto-failover
                end note
                stop
            end
        end

    else Package Not Found
        S3 --> OTAService: 404 Not Found
        deactivate S3
        OTAService --> Manager: 400 Bad Request\n{error: "Update package not found in S3"}
        deactivate OTAService
        note over Manager, S3 #FAILURE_COLOR
          FMEA: Update Package Not Found
          Severity: 3 (blocks deployment, no vehicle impact)
          Occurrence: 2 (configuration error)
          Detection: 1 (immediate)
          RPN: 6 (Low)
        end note
        stop
    end

else Validation Failed
    OTAService --> Manager: 400 Bad Request\n{error: "Invalid target filter syntax"}
    deactivate OTAService
    note over Manager, OTAService #FAILURE_COLOR
      FMEA: Invalid Campaign Configuration
      Severity: 3 (blocks deployment, no vehicle impact)
      Occurrence: 2 (human error)
      Detection: 1 (immediate)
      RPN: 6 (Low)
    end note
    stop
end

' Notification to Vehicle
OTAService -> S3: Generate presigned URL\n(expires: 1 hour)
activate S3
S3 --> OTAService: Presigned URL with credentials
deactivate S3

OTAService -> IoTCore: PUBLISH OTA notification
activate IoTCore
note right
Topic: v2c/v1/us-east-1/VIN-1HGBH41JXMN109186/ota/available
QoS: 2 (Exactly Once - Safety Critical)
Message Expiry: 86400s (24 hours)
Correlation ID: {uuid}
Payload: OTAUpdateAvailable {
  update_id: "ota-2025-001",
  version: "2.5.1",
  type: SECURITY_PATCH,
  priority: CRITICAL,
  size_bytes: 52428800,
  min_battery_level: 30,
  can_install_while_driving: false,
  estimated_install_time_sec: 600,
  download_url: {presigned_url},
  checksum_sha256: "a3f5b9c1e2d4...",
  signature: {hsm_signature},
  expires_at: 1633046400000
}
end note

alt MQTT Publish Failure
    IoTCore --X OTAService: Timeout (no PUBREC)
    OTAService -> OTAService: Retry MQTT publish\n(Attempt 2 of 3, exponential backoff)

    alt All Retries Failed
        deactivate IoTCore
        OTAService -> DLQ: Send notification to DLQ\n{vehicle_id, update_id, attempts: 3}
        activate DLQ
        DLQ --> OTAService: ACK
        deactivate DLQ

        OTAService -> Monitor: ALERT: MQTT broker unavailable
        activate Monitor
        Monitor -> Monitor: Check failure rate\n(5 failures in 1 minute)

        alt Alert Threshold Exceeded
            Monitor -> Engineer: PagerDuty: MQTT broker down\n(P1, immediate response)
            activate Engineer
            note over Engineer #CRITICAL_COLOR
              Manual Intervention Required:
              1. Check AWS IoT Core service status
              2. Verify IoT Core endpoint connectivity
              3. Check IoT Core throttling limits
              4. Review broker logs
              5. Replay DLQ messages after recovery
            end note
            deactivate Engineer
        end
        deactivate Monitor

        note over OTAService, IoTCore #CRITICAL_COLOR
          FMEA: MQTT Broker Failure
          Severity: 9 (affects ALL vehicles)
          Occurrence: 1 (rare, AWS outage)
          Detection: 1 (immediate)
          RPN: 9 (Low-Medium)
          Mitigation: Multi-region IoT Core, auto-failover
          Recovery: Replay from DLQ after restoration
        end note
    end
end

IoTCore --> OTAService: PUBREC (QoS 2 phase 1)
OTAService -> IoTCore: PUBREL (QoS 2 phase 2)
IoTCore --> OTAService: PUBCOMP (QoS 2 complete)
note right
**QoS 2 Exactly Once Guarantee**:
- Prevents duplicate installations
- Critical for safety-related updates
- 4-way handshake protocol
- Prevents over-the-air bricking
end note

IoTCore -> MQTT: Forward notification to vehicle
activate MQTT

alt Vehicle Offline
    MQTT --X MQTT: Vehicle not connected
    IoTCore -> IoTCore: Queue message in persistent session\n(cleanSession: false, sessionExpiryInterval: 3600s)
    note right
    **MQTT 5.0 Persistent Session**:
    - Messages queued up to 1 hour
    - Delivered when vehicle reconnects
    - QoS 2 state preserved
    - Prevents message loss
    end note

    note over MQTT #WARNING_COLOR
      FMEA: Vehicle Offline During Notification
      Severity: 5 (update delayed, not lost)
      Occurrence: 6 (common, vehicle parked/off)
      Detection: 2 (session expiry: 1 hour)
      RPN: 60 (Medium)
      Mitigation: Persistent session, retry when online
    end note

    MQTT -> IoTCore: CONNECT (vehicle reconnects later)
    IoTCore -> MQTT: CONNACK {sessionPresent: true}
    IoTCore -> MQTT: Deliver queued message
end

MQTT --> IoTCore: PUBREC
IoTCore -> MQTT: PUBREL
MQTT --> IoTCore: PUBCOMP
deactivate IoTCore

== Phase 2: Update Acceptance and Download ==

MQTT -> UpdateAgent: Forward OTA notification
activate UpdateAgent

UpdateAgent -> UpdateAgent: Check for concurrent updates\n(Max 1 active update)

alt Concurrent Update in Progress
    UpdateAgent -> MQTT: Reject update\n{status: REJECTED, reason: "UPDATE_IN_PROGRESS"}
    activate MQTT
    MQTT -> IoTCore: PUBLISH rejection
    activate IoTCore
    IoTCore -> OTAService: Deliver rejection
    OTAService -> DynamoDB: Update status = REJECTED
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent #WARNING_COLOR
      FMEA: Concurrent Update Conflict
      Severity: 4 (blocks update, retry later)
      Occurrence: 3 (multiple campaigns)
      Detection: 1 (immediate)
      RPN: 12 (Low)
      Mitigation: Campaign coordination, scheduling
    end note
    stop
end

UpdateAgent -> VehicleHSM: Verify digital signature\n(ECC P-256, SHA-256 hash)
activate VehicleHSM

alt Signature Valid
    VehicleHSM --> UpdateAgent: SIGNATURE_VALID
    deactivate VehicleHSM

    UpdateAgent -> Diagnostics: Get vehicle state
    activate Diagnostics
    Diagnostics --> UpdateAgent: {battery: 85%, ignition: false, speed: 0, parked: true}
    deactivate Diagnostics

    UpdateAgent -> Storage: Check available space\n(Required: 100MB + 50MB buffer)
    activate Storage

    alt Sufficient Storage
        Storage --> UpdateAgent: Available: 250MB
        deactivate Storage

        UpdateAgent -> MQTT: Accept update\n{status: ACCEPTED, scheduled_install_time: now()+3600}
        activate MQTT
        MQTT -> IoTCore: PUBLISH acceptance
        activate IoTCore
        IoTCore -> OTAService: Deliver acceptance
        OTAService -> DynamoDB: Update status = ACCEPTED
        deactivate IoTCore
        deactivate MQTT

    else Insufficient Storage
        Storage --> UpdateAgent: Available: 30MB (Required: 150MB)
        deactivate Storage

        UpdateAgent -> MQTT: Reject update\n{status: REJECTED, reason: "INSUFFICIENT_STORAGE"}
        activate MQTT
        MQTT -> IoTCore: PUBLISH rejection
        activate IoTCore
        IoTCore -> OTAService: Deliver rejection
        OTAService -> DynamoDB: Update status = REJECTED
        deactivate IoTCore
        deactivate MQTT

        note over UpdateAgent, Storage #FAILURE_COLOR
          FMEA: Insufficient Storage
          Severity: 6 (blocks update, vehicle functional)
          Occurrence: 4 (user data accumulation)
          Detection: 1 (immediate)
          RPN: 24 (Low-Medium)
          Mitigation: Storage cleanup recommendations, delta updates
        end note
        stop
    end

else Signature Invalid
    VehicleHSM --> UpdateAgent: SIGNATURE_INVALID
    deactivate VehicleHSM

    UpdateAgent -> MQTT: Reject update\n{status: REJECTED, reason: "SIGNATURE_VERIFICATION_FAILED"}
    activate MQTT
    MQTT -> IoTCore: PUBLISH rejection
    activate IoTCore
    IoTCore -> OTAService: Deliver rejection
    OTAService -> Monitor: CRITICAL: Signature verification failure
    activate Monitor
    Monitor -> Engineer: PagerDuty: Security alert - signature failure
    activate Engineer
    note over Engineer #CRITICAL_COLOR
      SECURITY INCIDENT:
      1. Investigate potential tampering
      2. Verify HSM key integrity
      3. Check S3 bucket access logs
      4. Validate update package integrity
      5. Incident response protocol
    end note
    deactivate Engineer
    deactivate Monitor
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent, VehicleHSM #CRITICAL_COLOR
      FMEA: Signature Verification Failure
      Severity: 10 (security vulnerability, prevents malicious updates)
      Occurrence: 1 (rare, configuration error or attack)
      Detection: 1 (immediate)
      RPN: 10 (Medium)
      Mitigation: HSM-based signing, certificate validation
      CRITICAL: Alert security team immediately
    end note
    stop
end

alt Battery Level Check
    UpdateAgent -> Diagnostics: Get battery level
    activate Diagnostics
    Diagnostics --> UpdateAgent: Battery: 25%
    deactivate Diagnostics

    alt Insufficient Battery (<30%)
        UpdateAgent -> MQTT: Defer update\n{status: DEFERRED, reason: "INSUFFICIENT_BATTERY", retry_after: 3600}
        activate MQTT
        MQTT -> IoTCore: PUBLISH deferral
        activate IoTCore
        IoTCore -> OTAService: Deliver deferral
        OTAService -> DynamoDB: Update status = DEFERRED
        deactivate IoTCore
        deactivate MQTT

        note over UpdateAgent #WARNING_COLOR
          FMEA: Insufficient Battery Level
          Severity: 6 (update delayed, safety precaution)
          Occurrence: 7 (common, vehicle parked)
          Detection: 1 (immediate)
          RPN: 42 (Medium)
          Mitigation: Wait for charging, user notification
        end note
        stop
    end
end

alt Vehicle State Check
    UpdateAgent -> Diagnostics: Get vehicle state
    activate Diagnostics
    Diagnostics --> UpdateAgent: {speed: 65 km/h, ignition: true}
    deactivate Diagnostics

    alt Vehicle In Use (not can_install_while_driving)
        UpdateAgent -> MQTT: Defer update\n{status: DEFERRED, reason: "VEHICLE_IN_USE"}
        activate MQTT
        MQTT -> IoTCore: PUBLISH deferral
        activate IoTCore
        IoTCore -> OTAService: Deliver deferral
        deactivate IoTCore
        deactivate MQTT

        note over UpdateAgent #WARNING_COLOR
          FMEA: Vehicle In Use During Installation
          Severity: 7 (safety precaution, correct behavior)
          Occurrence: 8 (common, user driving)
          Detection: 1 (immediate)
          RPN: 56 (Medium)
          Mitigation: Wait for vehicle parked, user scheduling
        end note
        stop
    end
end

' Download Phase
UpdateAgent -> UpdateAgent: Begin download\n(Chunk size: 1MB, supports resume)

UpdateAgent -> S3: GET Range: bytes=0-1048575\n(First 1MB chunk)
activate S3

alt S3 Available
    S3 --> UpdateAgent: 206 Partial Content (1MB)
    note right
    **HTTP Range Requests for Resume**:
    - Download in 1MB chunks
    - Support resume after interruption
    - Validate checksum incrementally
    - Bandwidth: ~1 Mbps (125 KB/s typical)
    end note

    UpdateAgent -> MQTT: Report progress 2%
    activate MQTT
    MQTT -> IoTCore: PUBLISH progress
    activate IoTCore
    IoTCore -> OTAService: Deliver progress
    OTAService -> DynamoDB: Update download_progress = 2%
    deactivate IoTCore
    deactivate MQTT

else S3 Service Unavailable
    S3 --> UpdateAgent: 503 Service Unavailable
    deactivate S3
    UpdateAgent -> UpdateAgent: Retry with exponential backoff\n(Attempt 2, wait 2s)

    alt All Retries Failed
        UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "S3_UNAVAILABLE"}
        activate MQTT
        MQTT -> IoTCore: PUBLISH failure
        activate IoTCore
        IoTCore -> OTAService: Deliver failure
        OTAService -> Monitor: Alert: S3 download failures
        deactivate IoTCore
        deactivate MQTT

        note over UpdateAgent, S3 #FAILURE_COLOR
          FMEA: S3 Repository Unavailable
          Severity: 8 (blocks downloads, affects multiple vehicles)
          Occurrence: 2 (AWS regional outage)
          Detection: 1 (immediate)
          RPN: 16 (Low-Medium)
          Mitigation: Multi-region S3, CloudFront CDN, retry logic
        end note
        stop
    end
end

' Continue downloading
UpdateAgent -> S3: GET Range: bytes=1048576-2097151 (chunk 2)
S3 --> UpdateAgent: 206 Partial Content

UpdateAgent -> S3: GET Range: bytes=10485760-11534335 (chunk 11)

alt Network Connection Lost During Download
    S3 --X UpdateAgent: Connection timeout
    deactivate S3

    UpdateAgent -> UpdateAgent: Connection lost at 10MB (20% complete)

    UpdateAgent -> MQTT: Report paused\n{status: PAUSED, progress: 20%, bytes_downloaded: 10485760}
    activate MQTT
    MQTT -> IoTCore: PUBLISH paused status
    activate IoTCore
    IoTCore -> OTAService: Deliver paused status
    OTAService -> DynamoDB: Update status = PAUSED, progress = 20%
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent, S3 #WARNING_COLOR
      FMEA: Network Connection Lost During Download
      Severity: 6 (download interrupted, can resume)
      Occurrence: 5 (network instability)
      Detection: 3 (connection timeout: 30s)
      RPN: 90 (Medium-High)
      Mitigation: HTTP Range resume support, auto-retry
      Recovery: Auto-resume when connection restored
    end note

    ' Connection restored
    UpdateAgent -> UpdateAgent: Wait for connection\n(retry every 60s, max 10 attempts)
    UpdateAgent -> UpdateAgent: Connection restored after 5 minutes

    UpdateAgent -> S3: GET Range: bytes=10485760-11534335\n(Resume from 10MB)
    activate S3
    S3 --> UpdateAgent: 206 Partial Content (resuming)

    UpdateAgent -> MQTT: Report resumed\n{status: IN_PROGRESS}
    activate MQTT
    MQTT -> IoTCore: PUBLISH resumed status
    activate IoTCore
    IoTCore -> OTAService: Deliver resumed status
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent #SUCCESS_COLOR
      Download Resume Successful:
      - No data loss
      - Continue from last successful chunk
      - HTTP Range requests enable efficient recovery
      - Total interruption time: 5 minutes
    end note
end

' Continue downloading to completion
UpdateAgent -> UpdateAgent: Continue downloading\n(Report progress every 10%)

UpdateAgent -> S3: GET Range: bytes=52420608-52428799 (final chunk)
S3 --> UpdateAgent: 206 Partial Content
deactivate S3

UpdateAgent -> UpdateAgent: Download complete (100%, 52428800 bytes)

' Checksum Validation
UpdateAgent -> UpdateAgent: Verify SHA-256 checksum

alt Checksum Mismatch
    UpdateAgent -> Storage: Delete corrupted file
    activate Storage
    Storage --> UpdateAgent: File deleted
    deactivate Storage

    UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "CHECKSUM_MISMATCH"}
    activate MQTT
    MQTT -> IoTCore: PUBLISH failure
    activate IoTCore
    IoTCore -> OTAService: Deliver failure
    OTAService -> Monitor: Alert: Checksum validation failure
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent #FAILURE_COLOR
      FMEA: Checksum Validation Failure
      Severity: 8 (prevents corrupted firmware, security)
      Occurrence: 2 (network corruption, S3 corruption)
      Detection: 1 (immediate)
      RPN: 16 (Low-Medium)
      Mitigation: Retry download, S3 integrity checks
      Security: Prevents corrupted firmware installation
    end note
    stop
end

UpdateAgent -> UpdateAgent: Checksum valid ✓

UpdateAgent -> MQTT: Report download complete\n{progress: 100%, status: DOWNLOADED}
activate MQTT
MQTT -> IoTCore: PUBLISH completion
activate IoTCore
IoTCore -> OTAService: Deliver completion
OTAService -> DynamoDB: Update status = DOWNLOADED, download_duration = 850s
deactivate IoTCore
deactivate MQTT

== Phase 3: Installation, Verification, and Rollback ==

UpdateAgent -> Diagnostics: Wait for optimal installation window
activate Diagnostics
Diagnostics --> UpdateAgent: Ignition off, parked, battery 85%
deactivate Diagnostics

UpdateAgent -> MQTT: Report installation started
activate MQTT
MQTT -> IoTCore: PUBLISH install start
activate IoTCore
IoTCore -> OTAService: Deliver install start
OTAService -> DynamoDB: Update status = INSTALLING
deactivate IoTCore
deactivate MQTT

' Pre-installation checks
UpdateAgent -> Diagnostics: Run pre-install checks
activate Diagnostics

alt Pre-Install Checks Passed
    Diagnostics --> UpdateAgent: All checks passed ✓\n(No critical DTCs, sufficient resources)
    deactivate Diagnostics

    UpdateAgent -> Storage: Identify target partition\n(Current: A, Target: B)
    activate Storage

    alt Partition B Available
        Storage --> UpdateAgent: Partition B available

        UpdateAgent -> MQTT: Report progress 10%\n(Backup creation)
        UpdateAgent -> Storage: Create backup of Partition A metadata
        Storage --> UpdateAgent: Backup created

        UpdateAgent -> MQTT: Report progress 20%\n(Extraction)
        UpdateAgent -> Storage: Extract package to Partition B

        alt Extraction Success
            Storage --> UpdateAgent: Extraction complete

            UpdateAgent -> MQTT: Report progress 40%\n(Applying update)
            UpdateAgent -> Storage: Write firmware to Partition B

            alt Flash Write Success
                Storage --> UpdateAgent: Firmware written successfully

                UpdateAgent -> MQTT: Report progress 70%
                UpdateAgent -> MQTT: Report progress 80%\n(Bootloader update)

                UpdateAgent -> Storage: Update boot flags
                note right
                {active_partition: B,
                fallback_partition: A,
                boot_attempts: 0,
                max_attempts: 3,
                watchdog_timeout: 120s}
                end note
                Storage --> UpdateAgent: Boot flags updated
                deactivate Storage

                UpdateAgent -> MQTT: Report progress 90%\n(Rebooting)
                activate MQTT
                MQTT -> IoTCore: PUBLISH progress
                activate IoTCore
                IoTCore -> OTAService: Deliver progress
                OTAService -> DynamoDB: Update status = REBOOTING
                deactivate IoTCore
                deactivate MQTT

                UpdateAgent -> UpdateAgent: Initiate system reboot

                note over UpdateAgent #WARNING_COLOR
                **SYSTEM REBOOT**
                Bootloader switches to Partition B
                Vehicle offline for ~60 seconds
                Watchdog timer: 120s per boot attempt
                end note

                ' Boot scenarios
                alt Boot Successful on New Firmware
                    UpdateAgent -> UpdateAgent: Boot from Partition B (version 2.5.1)
                    UpdateAgent -> UpdateAgent: Check boot_attempts counter\n(Attempt 1 of 3)
                    UpdateAgent -> UpdateAgent: Boot successful on Partition B ✓

                    UpdateAgent -> MQTT: Reconnect to AWS IoT Core
                    activate MQTT
                    MQTT -> IoTCore: CONNECT
                    activate IoTCore
                    IoTCore --> MQTT: CONNACK
                    deactivate IoTCore
                    deactivate MQTT

                    UpdateAgent -> MQTT: Report progress 95%\n(Post-install verification)

                    UpdateAgent -> Diagnostics: Run post-install verification
                    activate Diagnostics
                    note right
                    **Post-Install Verification Tests**:
                    1. CAN bus communication check
                    2. ECU firmware version validation
                    3. DTC scan (no new critical errors)
                    4. Sensor connectivity test
                    5. Network connectivity test
                    6. Feature smoke tests (6 total)
                    end note

                    alt All Verification Tests Passed
                        Diagnostics --> UpdateAgent: All tests passed ✓ (6/6)
                        deactivate Diagnostics

                        UpdateAgent -> Storage: Mark Partition B as stable
                        activate Storage
                        Storage --> UpdateAgent: Partition B marked stable
                        deactivate Storage

                        UpdateAgent -> MQTT: Report installation complete\n{status: SUCCESS, duration: 620s}
                        activate MQTT
                        MQTT -> IoTCore: PUBLISH completion
                        activate IoTCore
                        IoTCore -> OTAService: Deliver completion
                        OTAService -> DynamoDB: Update status = COMPLETED, install_duration = 620s
                        OTAService -> Monitor: Metric: Update successful
                        deactivate IoTCore
                        deactivate MQTT

                        note over UpdateAgent #SUCCESS_COLOR
                          **OTA Update Complete**
                          Total duration: ~25 minutes
                          - Notification: 5s
                          - Download: 14 minutes (850s)
                          - Installation: 10 minutes (620s)
                          - Verification: 12s
                          Vehicle running firmware 2.5.1 ✓
                        end note

                    else Verification Test Failed
                        Diagnostics --> UpdateAgent: FAILED: Test "CAN_BUS_COMMUNICATION" failed
                        deactivate Diagnostics

                        UpdateAgent -> Storage: Rollback to Partition A
                        activate Storage
                        Storage --> UpdateAgent: Boot flags reverted
                        deactivate Storage

                        UpdateAgent -> UpdateAgent: Reboot to Partition A

                        UpdateAgent -> MQTT: Report rollback\n{status: ROLLED_BACK, reason: "POST_INSTALL_VERIFICATION_FAILED"}
                        activate MQTT
                        MQTT -> IoTCore: PUBLISH rollback
                        activate IoTCore
                        IoTCore -> OTAService: Deliver rollback
                        OTAService -> DynamoDB: Update status = ROLLED_BACK
                        OTAService -> Monitor: Alert: Post-install verification failure
                        deactivate IoTCore
                        deactivate MQTT

                        note over UpdateAgent #FAILURE_COLOR
                          FMEA: Post-Install Verification Failure
                          Severity: 8 (new firmware fails tests, safe rollback)
                          Occurrence: 2 (firmware defect)
                          Detection: 1 (immediate)
                          RPN: 16 (Low-Medium)
                          Mitigation: Comprehensive test suite, A/B partitioning
                          Recovery: Vehicle operational on old firmware
                        end note
                    end

                else Boot Failure (New Firmware Crashes)
                    UpdateAgent -> UpdateAgent: Watchdog timeout (boot failed, attempt 1)
                    UpdateAgent -> UpdateAgent: Increment boot_attempts = 1
                    UpdateAgent -> UpdateAgent: Reboot and retry

                    UpdateAgent -> UpdateAgent: Boot from Partition B (Attempt 2)
                    UpdateAgent -> UpdateAgent: Watchdog timeout (failed again)
                    UpdateAgent -> UpdateAgent: Increment boot_attempts = 2

                    UpdateAgent -> UpdateAgent: Boot from Partition B (Attempt 3)
                    UpdateAgent -> UpdateAgent: Watchdog timeout (3rd failure)
                    UpdateAgent -> UpdateAgent: boot_attempts = 3 (MAX EXCEEDED)

                    UpdateAgent -> Storage: Automatic rollback triggered
                    activate Storage
                    Storage --> UpdateAgent: Boot flags reverted to Partition A
                    deactivate Storage

                    UpdateAgent -> UpdateAgent: Reboot from Partition A (old firmware)
                    UpdateAgent -> UpdateAgent: Boot successful (Partition A, v2.0.0)

                    UpdateAgent -> MQTT: Report rollback\n{status: ROLLED_BACK, reason: "BOOT_FAILURE", attempts: 3}
                    activate MQTT
                    MQTT -> IoTCore: PUBLISH rollback notification
                    activate IoTCore
                    IoTCore -> OTAService: Deliver rollback notification
                    OTAService -> DynamoDB: Update status = ROLLED_BACK
                    OTAService -> Monitor: CRITICAL: Update caused boot failures
                    activate Monitor
                    deactivate IoTCore
                    deactivate MQTT

                    OTAService -> OTAService: Check campaign failure threshold\n(5 of 10 rollbacks in canary)

                    alt Campaign Failure Threshold Exceeded
                        OTAService -> OTAService: Auto-pause campaign
                        OTAService -> DynamoDB: Update campaign status = PAUSED
                        activate DynamoDB
                        DynamoDB --> OTAService: Campaign paused
                        deactivate DynamoDB

                        Monitor -> Monitor: Aggregate rollback metrics
                        Monitor -> Engineer: PagerDuty: OTA campaign causing rollbacks\n(P1 incident)
                        activate Engineer
                        note over Engineer #CRITICAL_COLOR
                          Manual Investigation Required:
                          1. Review update package for defects
                          2. Analyze rollback logs from affected vehicles
                          3. Identify root cause (kernel panic, driver issue)
                          4. Run package through validation suite
                          5. Decide: Fix and resume OR cancel campaign
                          6. Notify fleet manager of pause
                        end note
                        deactivate Engineer
                    end
                    deactivate Monitor

                    note over UpdateAgent, Storage #CRITICAL_COLOR
                      FMEA: Boot Failure After Update
                      Severity: 9 (high impact, safe rollback prevents bricking)
                      Occurrence: 2 (firmware defect, rare)
                      Detection: 3 (3 boot attempts, ~6 minutes)
                      RPN: 54 (Medium-High)
                      Mitigation: A/B partitioning, automatic rollback, watchdog
                      Recovery: Vehicle remains operational on old firmware
                      Impact: User experiences 6-minute delay but vehicle functional
                    end note
                end

            else Flash Write Failure
                Storage --> UpdateAgent: ERROR: Flash write failed\n(Hardware error, bad blocks)
                deactivate Storage

                UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "FLASH_WRITE_FAILURE"}
                activate MQTT
                MQTT -> IoTCore: PUBLISH failure
                activate IoTCore
                IoTCore -> OTAService: Deliver failure
                OTAService -> Monitor: Alert: Flash write failure
                deactivate IoTCore
                deactivate MQTT

                note over UpdateAgent, Storage #CRITICAL_COLOR
                  FMEA: Flash Memory Write Failure
                  Severity: 8 (blocks update, Partition A still bootable)
                  Occurrence: 2 (hardware wear, rare)
                  Detection: 1 (immediate)
                  RPN: 16 (Low-Medium)
                  Mitigation: Partition A still bootable, vehicle operational
                  Action: Vehicle requires storage service/replacement
                  Impact: Cannot install updates until storage fixed
                end note
            end

        else Extraction Failed
            Storage --> UpdateAgent: ERROR: Extraction failed\n(Corrupted package, insufficient space)
            deactivate Storage

            UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "EXTRACTION_FAILED"}
            activate MQTT
            MQTT -> IoTCore: PUBLISH failure
            activate IoTCore
            IoTCore -> OTAService: Deliver failure
            deactivate IoTCore
            deactivate MQTT

            note over UpdateAgent #FAILURE_COLOR
              FMEA: Package Extraction Failure
              Severity: 6 (blocks update, vehicle functional)
              Occurrence: 4 (corrupted download despite checksum)
              Detection: 1 (immediate)
              RPN: 24 (Low-Medium)
              Mitigation: Retry download, verify package format
            end note
        end

    else Partition B Not Available
        Storage --> UpdateAgent: ERROR: Partition B corrupted
        deactivate Storage

        UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "PARTITION_UNAVAILABLE"}
        activate MQTT
        MQTT -> IoTCore: PUBLISH failure
        activate IoTCore
        IoTCore -> OTAService: Deliver failure
        OTAService -> Monitor: CRITICAL: A/B partition failure
        deactivate IoTCore
        deactivate MQTT

        note over UpdateAgent, Storage #CRITICAL_COLOR
          FMEA: A/B Partition Not Available
          Severity: 9 (blocks ALL future updates, vehicle may need factory reset)
          Occurrence: 1 (hardware failure, very rare)
          Detection: 1 (immediate)
          RPN: 9 (Low-Medium)
          Mitigation: Storage diagnostics, factory reset procedure
          CRITICAL: Vehicle may need dealership service
        end note
    end

else Critical DTC Found
    Diagnostics --> UpdateAgent: FAILED: Critical DTC P0601\n(Internal Control Module Memory)
    deactivate Diagnostics

    UpdateAgent -> MQTT: Report failure\n{status: FAILED, error: "CRITICAL_DTC_PRESENT"}
    activate MQTT
    MQTT -> IoTCore: PUBLISH failure
    activate IoTCore
    IoTCore -> OTAService: Deliver failure
    deactivate IoTCore
    deactivate MQTT

    note over UpdateAgent, Diagnostics #FAILURE_COLOR
      FMEA: Critical DTC Blocks Installation
      Severity: 7 (blocks update, indicates existing problem)
      Occurrence: 2 (pre-existing vehicle issue)
      Detection: 1 (immediate)
      RPN: 14 (Low-Medium)
      Action: Vehicle requires service before update
      Impact: OTA update deferred until DTC resolved
    end note
end

deactivate UpdateAgent

' Campaign Management
OTAService -> OTAService: Check campaign progress\n(Canary: 98/100 success, 2 rollbacks)

alt Canary Success Rate >= 95%
    OTAService -> OTAService: Canary validation passed ✓\n(98% success rate)
    OTAService -> OTAService: Proceed with full rollout\n(Remaining 4900 vehicles)
    OTAService -> DynamoDB: Update campaign status = ACTIVE, phase = FULL_ROLLOUT
    activate DynamoDB
    DynamoDB --> OTAService: Campaign updated
    deactivate DynamoDB

    note over OTAService #SUCCESS_COLOR
      **Canary Deployment Successful**
      Success rate: 98% (acceptable)
      Rollbacks: 2 (within threshold)
      Proceeding with full fleet rollout
      Estimated completion: 7 days (700 vehicles/day)
    end note

else Canary Failure Rate > 5%
    OTAService -> OTAService: Canary validation FAILED\n(85% success, 15 rollbacks)
    OTAService -> DynamoDB: Update campaign status = PAUSED
    activate DynamoDB
    DynamoDB --> OTAService: Campaign paused
    deactivate DynamoDB

    OTAService -> Monitor: ALERT: Campaign halted due to canary failures
    activate Monitor
    Monitor -> Engineer: PagerDuty: Campaign halted\n(P2, investigate within 2 hours)
    activate Engineer
    note over Engineer #FAILURE_COLOR
      Campaign Failure Investigation:
      1. Analyze rollback patterns (affected vehicle models)
      2. Review logs from failed vehicles
      3. Identify common failure modes
      4. Assess impact on remaining fleet
      5. Decision: Fix and resume OR cancel campaign
      6. Notify fleet manager and stakeholders
    end note
    deactivate Engineer
    deactivate Monitor

    note over OTAService #FAILURE_COLOR
      FMEA: Canary Deployment Failure
      Severity: 7 (affects 100 vehicles, fleet protected)
      Occurrence: 3 (firmware defect caught by canary)
      Detection: 1 (immediate, automated)
      RPN: 21 (Medium)
      Mitigation: Canary strategy limits blast radius
      Impact: Campaign halted before fleet-wide rollout
      Success: Canary prevented 4900 potential failures
    end note
end

== Monitoring and Alerting ==

Monitor -> Monitor: Aggregate metrics (every 30s)\n- Success rate\n- Download speed\n- Installation duration\n- Rollback rate\n- Campaign progress

alt Error Rate > 5%
    Monitor -> Monitor: Trigger alert:\n"High OTA failure rate"
    Monitor -> Engineer: Slack notification + PagerDuty (P2)
else Rollback Rate > 3%
    Monitor -> Monitor: Trigger alert:\n"High rollback rate"
    Monitor -> Engineer: PagerDuty (P1)
else Campaign Stalled (>24h no progress)
    Monitor -> Monitor: Trigger alert:\n"Campaign stalled"
    Monitor -> Engineer: Email + Slack
end

@enduml
```

---

## FMEA Table

| Failure Mode | Severity | Occurrence | Detection | RPN | Mitigation | Status |
|--------------|----------|------------|-----------|-----|------------|--------|
| Invalid campaign configuration | 3 | 2 | 1 | 6 | Input validation, schema checks | ✅ Acceptable |
| Update package not found in S3 | 3 | 2 | 1 | 6 | Pre-deployment verification, CI/CD checks | ✅ Acceptable |
| **HSM signing service unavailable** | **9** | **1** | **1** | **9** | Multi-region HSM, health checks, auto-failover | ⚠️ **Low-Medium** |
| **MQTT broker failure** | **9** | **1** | **1** | **9** | Multi-region IoT Core, auto-failover, DLQ | ⚠️ **Low-Medium** |
| Vehicle offline during notification | 5 | 6 | 2 | 60 | Persistent session, retry when online | ✅ Acceptable |
| Concurrent update conflict | 4 | 3 | 1 | 12 | Campaign coordination, scheduling | ✅ Acceptable |
| **Signature verification failure** | **10** | **1** | **1** | **10** | HSM-based signing, certificate validation | ⚠️ **Medium - Security Critical** |
| **Insufficient battery level** | **6** | **7** | **1** | **42** | Wait for charging, user notification, scheduling | ⚠️ **Medium** |
| Insufficient storage space | 6 | 4 | 1 | 24 | Storage cleanup, delta updates, user guidance | ✅ Acceptable |
| **Vehicle in use during installation** | **7** | **8** | **1** | **56** | Wait for parked state, user scheduling, smart timing | ⚠️ **Medium** |
| S3 repository unavailable | 8 | 2 | 1 | 16 | Multi-region S3, CloudFront CDN, retry logic | ✅ Acceptable |
| **Network connection lost during download** | **6** | **5** | **3** | **90** | HTTP Range resume, auto-retry, progress tracking | ⚠️ **Medium-High** |
| Checksum validation failure | 8 | 2 | 1 | 16 | Retry download, S3 integrity checks | ✅ Acceptable |
| Critical DTC blocks installation | 7 | 2 | 1 | 14 | Pre-install checks, service requirement | ✅ Acceptable |
| Package extraction failure | 6 | 4 | 1 | 24 | Retry download, format validation | ✅ Acceptable |
| Flash memory write failure | 8 | 2 | 1 | 16 | A/B partitioning, storage diagnostics | ✅ Acceptable |
| A/B partition not available | 9 | 1 | 1 | 9 | Storage diagnostics, factory reset procedure | ⚠️ **Low-Medium** |
| Post-install verification failure | 8 | 2 | 1 | 16 | Comprehensive test suite, automatic rollback | ✅ Acceptable |
| **Boot failure after update** | **9** | **2** | **3** | **54** | A/B partitioning, automatic rollback, watchdog | ⚠️ **Medium-High** |
| Canary deployment failure | 7 | 3 | 1 | 21 | Canary strategy limits blast radius | ✅ Acceptable |
| Campaign stalled (no progress >24h) | 5 | 3 | 5 | 75 | Progress monitoring, auto-escalation | ⚠️ **Medium** |
| Version conflict (downgrade attempt) | 4 | 2 | 1 | 8 | Version comparison, downgrade protection | ✅ Acceptable |
| Certificate expiration | 8 | 1 | 1 | 8 | Certificate rotation, expiry monitoring | ✅ Acceptable |
| Download timeout (>1 hour) | 6 | 4 | 6 | 144 | Chunk-based download, resume support | ❌ **High - Action Required** |

**RPN Severity Levels:**
- **Low (RPN < 20):** Acceptable, monitor
- **Medium (RPN 20-100):** Mitigation recommended
- **High (RPN > 100):** Action required, prioritize

---

## Action Items

### High Priority (RPN > 100)

1. **Download Timeout (RPN 144)**
   - **Current:** 1-hour timeout for 50MB download causes abandonment
   - **Action:** Implement adaptive chunk sizing based on network speed
   - **Additional:** Add background download support, pause/resume UI
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q1 2026

### Medium Priority (RPN 20-100)

2. **Network Connection Lost During Download (RPN 90)**
   - **Current:** HTTP Range resume requires manual retry
   - **Action:** Implement automatic resume with exponential backoff
   - **Budget:** Improve network quality monitoring
   - **Owner:** Connectivity Team
   - **Deadline:** Q2 2026

3. **Campaign Stalled (RPN 75)**
   - **Current:** Manual investigation required for stalled campaigns
   - **Action:** Implement auto-escalation after 24 hours
   - **Additional:** Add campaign health dashboard with real-time metrics
   - **Owner:** OTA Service Team
   - **Deadline:** Q2 2026

4. **Vehicle In Use During Installation (RPN 56)**
   - **Current:** Installation deferred until vehicle parked
   - **Action:** Implement smart scheduling with ML-based optimal window prediction
   - **Additional:** User scheduling interface in mobile app
   - **Owner:** UX Team + Backend Team
   - **Deadline:** Q2 2026

5. **Boot Failure After Update (RPN 54)**
   - **Current:** 3 boot attempts before rollback (6-minute delay)
   - **Action:** Reduce boot attempts to 2, improve watchdog timeout logic
   - **Additional:** Add pre-flight simulation in staging environment
   - **Owner:** Bootloader Team
   - **Deadline:** Q3 2026

6. **Insufficient Battery Level (RPN 42)**
   - **Current:** Update deferred until 30% battery
   - **Action:** Implement opportunistic charging detection, install during charging
   - **Budget:** SMS notification when vehicle reaches charge threshold
   - **Owner:** Power Management Team
   - **Deadline:** Q3 2026

---

## Test Plan

### Unit Tests

```python
def test_signature_verification_invalid():
    """Verify invalid signature rejected"""
    update = OTAUpdateAvailable(
        update_id="test",
        signature="invalid_signature"
    )
    result = vehicle_hsm.verify_signature(update)
    assert result == SignatureStatus.INVALID
    assert update_agent.accept_update(update) == False
    assert rejection_reason == "SIGNATURE_VERIFICATION_FAILED"

def test_download_resume_after_interruption():
    """Verify download resumes from correct byte offset"""
    download_manager.start_download(update_url, size=52428800)
    download_manager.download_chunks(0, 10485760)  # Download 10MB

    # Simulate network interruption
    network.disconnect()
    assert download_manager.bytes_downloaded == 10485760

    # Restore connection and resume
    network.reconnect()
    download_manager.resume_download()

    # Verify resume from correct offset
    assert last_http_request.headers["Range"] == "bytes=10485760-11534335"

def test_automatic_rollback_after_boot_failure():
    """Verify automatic rollback after 3 failed boots"""
    bootloader.set_active_partition("B")
    bootloader.set_boot_attempts(0)

    # Simulate 3 failed boots
    for i in range(3):
        result = bootloader.attempt_boot()
        assert result == BootStatus.FAILED
        assert bootloader.boot_attempts == i + 1

    # Verify automatic rollback triggered
    assert bootloader.active_partition == "A"
    assert bootloader.boot_attempts == 0
    assert vehicle_boots_successfully()
```

### Integration Tests

```python
def test_end_to_end_ota_update():
    """Test complete OTA update flow"""
    # Phase 1: Create campaign
    campaign = ota_service.create_campaign(
        version="2.5.1",
        rollout_strategy=RolloutStrategy.CANARY,
        canary_size=100
    )
    assert campaign.status == CampaignStatus.CREATED

    # Phase 2: Notify vehicle
    ota_service.notify_vehicles(campaign)
    notification = vehicle.mqtt_client.receive_message()
    assert notification.update_id == campaign.update_id

    # Phase 3: Vehicle accepts and downloads
    vehicle.update_agent.process_notification(notification)
    assert vehicle.update_agent.status == UpdateStatus.DOWNLOADING

    # Wait for download completion
    wait_for_download_complete(timeout=900)
    assert vehicle.storage.get_file_size("update.bin") == 52428800

    # Phase 4: Install and verify
    vehicle.update_agent.install_update()
    vehicle.reboot()
    assert vehicle.firmware_version == "2.5.1"
    assert vehicle.diagnostics.run_post_install_checks() == True

def test_rollback_on_verification_failure():
    """Test automatic rollback when verification fails"""
    original_version = vehicle.firmware_version

    # Install defective update
    vehicle.update_agent.install_update(defective_update)
    vehicle.reboot()

    # Post-install verification fails
    assert vehicle.diagnostics.run_post_install_checks() == False

    # Verify automatic rollback
    assert vehicle.firmware_version == original_version
    assert vehicle.bootloader.active_partition == "A"
    assert mqtt_messages.contains("status: ROLLED_BACK")
```

### Chaos Tests

```python
def test_network_interruption_during_download():
    """Verify graceful handling of network interruptions"""
    download_manager.start_download(update_url, size=52428800)

    # Download 20%, then disconnect
    download_manager.download_chunks(0, 10485760)
    with network_partition():
        assert download_manager.status == DownloadStatus.PAUSED
        assert mqtt_messages.contains("status: PAUSED")

    # Verify auto-resume after connection restored
    wait_for_network_reconnection()
    assert download_manager.status == DownloadStatus.IN_PROGRESS
    wait_for_download_complete()
    assert download_manager.bytes_downloaded == 52428800

def test_mqtt_broker_partition():
    """Verify graceful degradation during broker partition"""
    with mqtt_broker_partition():
        response = ota_service.notify_vehicle(vehicle_id)
        assert response.status == 503
        assert "Service Unavailable" in response.message

    # Verify DLQ contains failed notification
    assert dlq_messages.contains(vehicle_id)

    # Verify replay after recovery
    mqtt_broker_recover()
    ota_service.replay_dlq_messages()
    notification = vehicle.mqtt_client.receive_message()
    assert notification.update_id is not None

def test_s3_unavailability():
    """Verify download retry logic when S3 unavailable"""
    with s3_service_unavailable():
        result = vehicle.update_agent.download_update(presigned_url)
        assert result.status == DownloadStatus.FAILED
        assert result.error == "S3_UNAVAILABLE"
        assert result.retry_count == 3
```

---

## Performance Benchmarks

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| Campaign Creation Time | <5s | 3.2s | ✅ Pass |
| Notification Delivery (QoS 2) | <10s | 5.1s | ✅ Pass |
| Download Speed (50MB) | >100 KB/s | 125 KB/s | ✅ Pass |
| Download Time (50MB) | <15 min | 14 min | ✅ Pass |
| Installation Time | <12 min | 10.3 min | ✅ Pass |
| Rollback Time | <5 min | 3.8 min | ✅ Pass |
| Success Rate (Canary) | >95% | 98.0% | ✅ Pass |
| Success Rate (Full Rollout) | >97% | 98.5% | ✅ Pass |
| Rollback Rate | <3% | 2.0% | ✅ Pass |

---

## Monitoring and Alerts

### Metrics

```
# Prometheus-style metrics
v2c_ota_campaign_status{campaign_id, version, phase="canary|full_rollout", status="active|paused|completed"}
v2c_ota_update_status{campaign_id, vehicle_id, status="pending|downloading|installing|completed|failed|rolled_back"}
v2c_ota_download_speed_bps{campaign_id, vehicle_id, percentile="50|95|99"}
v2c_ota_download_duration_seconds{campaign_id, vehicle_id, percentile="50|95|99"}
v2c_ota_installation_duration_seconds{campaign_id, vehicle_id, percentile="50|95|99"}
v2c_ota_rollback_count{campaign_id, reason}
v2c_ota_signature_verification_failures_total{campaign_id}
v2c_ota_checksum_failures_total{campaign_id}
v2c_hsm_signing_latency_seconds{operation="sign_metadata"}
v2c_s3_download_errors_total{bucket, reason}
```

### Alerts

```yaml
- alert: HighOTARollbackRate
  expr: rate(v2c_ota_rollback_count[10m]) > 0.03
  for: 10m
  severity: P1
  description: "OTA rollback rate {{ $value }}% exceeds 3% threshold"

- alert: HSMSigningServiceDown
  expr: up{job="hsm-signing"} == 0
  for: 1m
  severity: P1
  description: "HSM signing service {{ $labels.instance }} is down"

- alert: OTACampaignStalled
  expr: time() - v2c_ota_campaign_last_update_timestamp > 86400
  for: 30m
  severity: P2
  description: "OTA campaign {{ $labels.campaign_id }} has no progress for >24h"

- alert: SignatureVerificationFailure
  expr: rate(v2c_ota_signature_verification_failures_total[5m]) > 0
  for: 1m
  severity: P0
  description: "SECURITY: Signature verification failures detected in campaign {{ $labels.campaign_id }}"

- alert: HighDownloadTimeout
  expr: rate(v2c_ota_update_status{status="failed", reason="DOWNLOAD_TIMEOUT"}[1h]) > 0.05
  for: 30m
  severity: P2
  description: "Download timeout rate {{ $value }}% exceeds 5% threshold"

- alert: CanaryDeploymentFailure
  expr: (v2c_ota_update_status{phase="canary", status="failed|rolled_back"} / v2c_ota_update_status{phase="canary"}) > 0.05
  for: 5m
  severity: P1
  description: "Canary deployment failure rate {{ $value }}% exceeds 5% threshold, campaign auto-paused"
```

---

## Security Considerations

### Signature Verification

All OTA updates MUST be digitally signed with ECC P-256 using AWS CloudHSM:

1. **Signing Process:**
   - OTA Service generates SHA-256 hash of update package
   - CloudHSM signs hash with private key (stored in HSM, never exported)
   - Signature included in OTA notification message

2. **Verification Process:**
   - Vehicle HSM/TPM verifies signature using public key
   - Public key embedded in vehicle firmware (root of trust)
   - Signature mismatch triggers immediate rejection and security alert

3. **Certificate Rotation:**
   - Signing certificates rotated every 12 months
   - New certificates distributed via previous signed update
   - Grace period: 30 days overlap for old and new certificates

### Threat Model

| Threat | Mitigation | Residual Risk |
|--------|------------|---------------|
| Malicious update injection | HSM-based signing, signature verification | Low |
| Man-in-the-middle attack | TLS 1.3, certificate pinning, MQTT over TLS | Low |
| Replay attack (old firmware) | Version comparison, downgrade protection, timestamp validation | Low |
| Update package tampering | SHA-256 checksum, signature verification | Low |
| Certificate compromise | HSM storage (non-exportable), rotation policy | Medium |
| Physical access attack | TPM-based verification, secure boot | Medium |

---

## References

- [OTA Notification Diagram](../sequence-diagrams/03a-ota-notification.png)
- [OTA Download Diagram](../sequence-diagrams/03b-ota-download.png)
- [OTA Installation Diagram](../sequence-diagrams/03c-ota-installation.png)
- [Topic Naming Convention](../standards/TOPIC_NAMING_CONVENTION.md)
- [QoS Selection Guide](../standards/QOS_SELECTION_GUIDE.md)
- [API and Protocol Reference](../API_AND_PROTOCOL_REFERENCE.md)
- [ISO 21434 Cybersecurity Engineering](https://www.iso.org/standard/70918.html)
- [MQTT 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

---

## Revision History

| Version | Date       | Author                | Changes                          |
|---------|------------|-----------------------|----------------------------------|
| 1.0     | 2025-10-14 | V2C Architecture Team | Initial FMEA analysis for OTA updates |

---

## Approval

| Role                          | Name                | Signature | Date       |
|-------------------------------|---------------------|-----------|------------|
| Architecture Lead             | ___________________ | _________ | __________ |
| Vehicle Platform Lead         | ___________________ | _________ | __________ |
| Quality Assurance Lead        | ___________________ | _________ | __________ |
| ISO 21434 Compliance Officer  | ___________________ | _________ | __________ |
| Security Lead                 | ___________________ | _________ | __________ |

---

**Document Classification:** Internal - Engineering Documentation
**Security Level:** Confidential
**Distribution:** V2C Development Team, QA Team, Vehicle Engineering, Operations, Security Team
