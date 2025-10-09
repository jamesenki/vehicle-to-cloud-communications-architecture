# FMEA: Remote Door Lock Command

**Feature:** Remote Door Lock/Unlock via Mobile App
**Version:** 1.0
**Last Updated:** 2025-10-09
**Owner:** V2C Architecture Team
**Compliance:** ISO 21434, MQTT 5.0, Project FMEA Standards

---

## Executive Summary

This document provides a comprehensive Failure Mode and Effects Analysis (FMEA) for the Remote Door Lock command feature in the Vehicle-to-Cloud (V2C) communications system. The analysis covers the complete end-to-end flow from mobile app initiation to vehicle execution and response.

### Key Metrics

| Metric                        | Value          |
|-------------------------------|----------------|
| Expected Success Rate         | 99.5%          |
| P95 Latency (Happy Path)      | 1.2 seconds    |
| P99 Latency (with Retries)    | 3.5 seconds    |
| Maximum Timeout               | 30 seconds     |
| MTTR (Mean Time to Recovery)  | <5 minutes     |
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

title FMEA-Ready Sequence: Remote Door Lock Command
skinparam backgroundColor #FEFEFE
skinparam responseMessageBelowArrow true
skinparam sequenceMessageAlign center

' Define all participants
actor "Mobile User" as User #LightBlue
participant "Mobile App\n[iOS/Android]" as App #LightGreen
participant "API Gateway\n[Rate Limit: 100/min]\n[Timeout: 5s]" as Gateway #LightYellow
participant "Auth Service\n[JWT, mTLS]\n[Timeout: 3s]" as Auth #LightCyan
database "PostgreSQL\n[Isolation: READ_COMMITTED]\n[Connection Pool: 50]" as DB #LightGray
participant "Command Service\n[Timeout: 25s]\n[Circuit Breaker]" as CmdService #LightGreen
participant "Vehicle State Service\n[Cache: Redis, TTL=60s]" as VehicleState #LightSalmon
queue "MQTT Broker\n[AWS IoT Core]\n[QoS: 0,1,2]" as Broker #LightPink
participant "Vehicle TCU\n[Always Connected]\n[Idempotency Cache]" as Vehicle #LightBlue
participant "Door Lock ECU\n[CAN Bus]" as DoorECU #LightGray
queue "Dead Letter Queue\n[SQS, Retention: 14d]" as DLQ #FAILURE_COLOR
participant "Monitoring\n[CloudWatch/Datadog]\n[Alert Threshold: 5 failures/min]" as Monitor #Orange
participant "On-Call Engineer" as Engineer #Red

== Authentication Phase ==

User -> App: Tap "Lock Doors" button
activate App

App -> App: Check cached auth token
note right
  Token TTL: 1 hour
  Refresh if <5min remaining
end note

alt Token Valid
    App -> App: Use cached token
else Token Expired or Missing
    App -> Auth: POST /auth/token\n{username, password}
    activate Auth

    Auth -> DB: BEGIN TRANSACTION (READ_COMMITTED)
    activate DB

    alt Valid Credentials
        DB --> Auth: User credentials validated
        Auth -> Auth: Generate JWT token\n(exp: 1 hour, iss: v2c-auth)
        Auth -> DB: INSERT INTO audit_log\n(user_id, action, timestamp)
        Auth -> DB: COMMIT
        deactivate DB
        Auth --> App: 200 OK\n{token, expires_in: 3600}
        deactivate Auth

    else Invalid Credentials
        DB --> Auth: No matching credentials
        Auth -> DB: ROLLBACK
        deactivate DB
        Auth -> Monitor: Log failed login attempt\n(user_id, ip_address)
        Auth --> App: 401 Unauthorized\n{error: "Invalid credentials"}
        deactivate Auth
        App --> User: ERROR: Invalid username/password
        note over User, App #FAILURE_COLOR
          FMEA: Authentication Failure
          Severity: 3 (blocks user, no safety impact)
          Occurrence: 2 (common user error)
          Detection: 1 (immediate feedback)
          RPN: 6 (Low - acceptable)
        end note
        stop

    else Auth Service Timeout (>3s)
        Auth --X App: Timeout (no response)
        deactivate Auth
        App -> App: Retry with exponential backoff\n(Attempt 2 of 3, wait 2s)

        alt Retry Successful
            App -> Auth: POST /auth/token (retry)
            activate Auth
            Auth --> App: 200 OK {token}
            deactivate Auth
        else All Retries Failed
            App -> Monitor: CRITICAL: Auth service down
            App --> User: ERROR: Service unavailable,\ntry again later
            note over User, App #CRITICAL_COLOR
              FMEA: Auth Service Unavailable
              Severity: 5 (blocks all operations)
              Occurrence: 1 (rare, deployment/outage)
              Detection: 1 (immediate)
              RPN: 5 (Low)
              Mitigation: Auto-scaling, health checks
            end note
            stop
        end
    end
end

== Rate Limiting Check ==

App -> Gateway: POST /api/v1/vehicle/{VIN}/commands/lock-doors\nAuthorization: Bearer {token}\nX-Request-ID: {uuid}\nX-Idempotency-Key: {uuid}
activate Gateway

Gateway -> Gateway: Check rate limit\n(Token bucket: 100 req/min per user)

alt Rate Limit Exceeded
    Gateway --> App: 429 Too Many Requests\nRetry-After: 30\nX-RateLimit-Remaining: 0
    deactivate Gateway
    App --> User: ERROR: Too many requests,\nplease wait 30 seconds
    note over User, Gateway #WARNING_COLOR
      FMEA: Rate Limit Exceeded
      Severity: 3 (temporary block)
      Occurrence: 2 (user error or malicious)
      Detection: 1 (immediate)
      RPN: 6 (Low)
      Mitigation: Educate users, implement client-side throttling
    end note
    stop

else Within Rate Limit
    Gateway -> Gateway: Decrement token bucket\n(Remaining: 99)

    Gateway -> Auth: Validate JWT token\n(signature, expiry, issuer)
    activate Auth

    alt Token Valid
        Auth --> Gateway: 200 OK {user_id, permissions}
        deactivate Auth

    else Token Invalid or Expired
        Auth --> Gateway: 401 Unauthorized
        deactivate Auth
        Gateway --> App: 401 Unauthorized\n{error: "Token expired"}
        deactivate Gateway
        App -> App: Clear cached token
        App --> User: Session expired, please log in
        stop
    end
end

== Command Service Processing ==

Gateway -> CmdService: POST /internal/commands/lock-doors\nX-User-ID: {user_id}\nX-Request-ID: {uuid}\nX-Idempotency-Key: {uuid}\n{vehicle_id: "VIN12345"}
activate CmdService

CmdService -> CmdService: Check idempotency cache\n(Redis, key: idempotency_key)

alt Already Processed (Duplicate Request)
    CmdService -> CmdService: Retrieve cached response
    CmdService --> Gateway: 200 OK (cached)\n{success: true, message: "Already processed"}
    deactivate CmdService
    Gateway --> App: 200 OK
    deactivate Gateway
    App --> User: SUCCESS: Doors locked
    note over CmdService #SUCCESS_COLOR
      FMEA: Duplicate Request Handled
      Idempotency prevents double execution
      No failure mode
    end note
    stop

else Not Processed Yet
    CmdService -> VehicleState: GET /vehicle/{VIN}/state
    activate VehicleState

    VehicleState -> VehicleState: Check Redis cache\n(key: vehicle:VIN12345:state, TTL: 60s)

    alt Cache Hit
        VehicleState --> CmdService: 200 OK (cached)\n{connected: true, state: "IGNITION_ON"}
        deactivate VehicleState

    else Cache Miss
        VehicleState -> DB: SELECT * FROM vehicle_state\nWHERE vehicle_id = 'VIN12345'
        activate DB

        alt Vehicle Found
            DB --> VehicleState: {connected: true, last_seen: "2025-10-09T10:30:00Z"}
            deactivate DB
            VehicleState -> VehicleState: Cache result in Redis (TTL: 60s)
            VehicleState --> CmdService: 200 OK\n{connected: true}
            deactivate VehicleState

        else Vehicle Not Found
            DB --> VehicleState: No rows
            deactivate DB
            VehicleState --> CmdService: 404 Not Found
            deactivate VehicleState
            CmdService --> Gateway: 404 Not Found\n{error: "Vehicle not found"}
            deactivate CmdService
            Gateway --> App: 404 Not Found
            deactivate Gateway
            App --> User: ERROR: Vehicle not registered
            note over CmdService, DB #FAILURE_COLOR
              FMEA: Vehicle Not Provisioned
              Severity: 4 (blocks feature for user)
              Occurrence: 1 (rare, provisioning issue)
              Detection: 1 (immediate)
              RPN: 4 (Low)
            end note
            stop
        end
    end

    alt Vehicle Offline
        CmdService -> CmdService: Check last_seen timestamp\n(>5 minutes = offline)
        CmdService --> Gateway: 503 Service Unavailable\n{error: "Vehicle offline", code: "DEVICE_OFFLINE"}
        deactivate CmdService
        Gateway --> App: 503 Service Unavailable
        deactivate Gateway
        App --> User: ERROR: Vehicle is offline.\nTry again when vehicle is powered on.
        note over CmdService, Vehicle #WARNING_COLOR
          FMEA: Vehicle Offline
          Severity: 6 (user expectation not met)
          Occurrence: 8 (common, vehicle off/parked)
          Detection: 2 (5-minute delay to detect)
          RPN: 96 (Medium-High)
          Mitigation: SMS shoulder tap (costs $0.05/msg)
          Decision: No auto-retry for cost control
        end note
        stop

    else Vehicle Connected
        CmdService -> DB: BEGIN TRANSACTION (READ_COMMITTED)
        activate DB

        CmdService -> DB: INSERT INTO commands\n(id, vehicle_id, type, status, created_at)\nVALUES (uuid, 'VIN12345', 'lock_doors', 'PENDING', now())
        DB --> CmdService: 1 row inserted

        CmdService -> DB: COMMIT
        deactivate DB

        ' Publish to MQTT
        CmdService -> Broker: PUBLISH\nTopic: v2c/v1/us-east/VIN12345/commands/lock-doors/request\nQoS: 1 (At Least Once)\nMessage Expiry: 30s\nResponse Topic: v2c/v1/us-east/VIN12345/commands/lock-doors/response\nCorrelation ID: {uuid}\nPayload: {idempotency_key, command: "lock_doors", timestamp}
        activate Broker

        alt MQTT Publish Success
            Broker --> CmdService: PUBACK (QoS 1 acknowledgment)
            note right of Broker
              Broker acknowledges receipt
              Does NOT guarantee vehicle delivery
            end note

        else MQTT Broker Timeout (>2s)
            Broker --X CmdService: Timeout (no PUBACK)
            CmdService -> CmdService: Retry MQTT publish\n(Attempt 2 of 3, exponential backoff)

            alt Retry Successful
                CmdService -> Broker: PUBLISH (retry)
                Broker --> CmdService: PUBACK
            else All Retries Failed
                deactivate Broker
                CmdService -> DLQ: Send command to DLQ\n{vehicle_id, command, attempts: 3}
                activate DLQ
                DLQ --> CmdService: ACK
                deactivate DLQ

                CmdService -> DB: UPDATE commands SET status = 'FAILED',\nerror = 'MQTT broker unavailable'\nWHERE id = {uuid}
                activate DB
                DB --> CmdService: 1 row updated
                deactivate DB

                CmdService -> Monitor: ALERT: MQTT broker unavailable\n(P1 incident)
                activate Monitor
                Monitor -> Monitor: Check failure rate\n(5 failures in 1 minute)

                alt Alert Threshold Exceeded
                    Monitor -> Engineer: PagerDuty: MQTT broker down\n(P1, immediate response required)
                    activate Engineer
                    note over Engineer #CRITICAL_COLOR
                      Manual Intervention Required
                      1. Check AWS IoT Core status
                      2. Verify network connectivity
                      3. Review broker logs
                      4. Restart broker if needed
                      5. Replay DLQ messages after recovery
                    end note
                    deactivate Engineer
                end
                deactivate Monitor

                CmdService --> Gateway: 503 Service Unavailable\n{error: "MQTT broker unavailable"}
                deactivate CmdService
                Gateway --> App: 503 Service Unavailable
                deactivate Gateway
                App --> User: ERROR: Service temporarily unavailable
                note over CmdService, Broker #CRITICAL_COLOR
                  FMEA: MQTT Broker Failure
                  Severity: 8 (affects all vehicles)
                  Occurrence: 1 (rare, AWS outage)
                  Detection: 1 (immediate)
                  RPN: 8 (Low-Medium)
                  Mitigation: Multi-region broker, auto-failover
                end note
                stop
            end
        end

        ' Wait for vehicle response
        Broker -> Vehicle: PUBLISH\nTopic: v2c/v1/us-east/VIN12345/commands/lock-doors/request\nQoS: 1\nPayload: {idempotency_key, command: "lock_doors"}
        activate Vehicle

        Vehicle -> Vehicle: Check idempotency cache\n(LRU cache, 1000 entries, 24h TTL)

        alt Already Processed (Duplicate from QoS 1 retry)
            Vehicle -> Vehicle: Retrieve cached response
            Vehicle -> Broker: PUBLISH\nTopic: v2c/v1/us-east/VIN12345/commands/lock-doors/response\nQoS: 1\nCorrelation ID: {uuid}\nPayload: {success: true, cached: true}
            Broker -> Vehicle: PUBACK
            note over Vehicle #SUCCESS_COLOR
              QoS 1 Duplicate Handled
              Idempotency prevents double lock
            end note

        else Not Processed
            Vehicle -> Vehicle: Check vehicle state\n(ignition, speed, doors)

            alt Invalid State (Speed > 5 km/h)
                Vehicle -> Broker: PUBLISH response\nPayload: {success: false, error: "INVALID_STATE",\nmessage: "Cannot lock doors while driving"}
                Broker -> Vehicle: PUBACK
                Broker --> CmdService: Response message
                deactivate Vehicle

                CmdService -> DB: UPDATE commands SET status = 'REJECTED',\nerror = 'INVALID_STATE'\nWHERE id = {uuid}
                activate DB
                DB --> CmdService: 1 row updated
                deactivate DB

                CmdService --> Gateway: 400 Bad Request\n{error: "Cannot lock doors while driving"}
                deactivate CmdService
                Gateway --> App: 400 Bad Request
                deactivate Gateway
                App --> User: ERROR: Cannot lock doors while driving
                note over Vehicle #WARNING_COLOR
                  FMEA: Invalid Vehicle State
                  Severity: 3 (safety precaution, correct behavior)
                  Occurrence: 3 (user error)
                  Detection: 1 (immediate)
                  RPN: 9 (Low)
                end note
                stop

            else Valid State (Parked or Low Speed)
                Vehicle -> DoorECU: CAN Message\n0x2A5: LOCK_ALL_DOORS
                activate DoorECU

                alt Door Lock Success
                    DoorECU -> DoorECU: Actuate door locks
                    DoorECU --> Vehicle: CAN Response\n0x2A6: STATUS_LOCKED
                    deactivate DoorECU

                    Vehicle -> Vehicle: Update local state\n(door_lock_state: LOCKED)
                    Vehicle -> Vehicle: Cache response\n(idempotency_key → response)

                    Vehicle -> Broker: PUBLISH response\nTopic: response topic\nQoS: 1\nPayload: {success: true, timestamp: now()}
                    Broker -> Vehicle: PUBACK
                    Broker --> CmdService: Response message
                    deactivate Vehicle

                    CmdService -> DB: UPDATE commands SET status = 'SUCCESS',\ncompleted_at = now()\nWHERE id = {uuid}
                    activate DB
                    DB --> CmdService: 1 row updated
                    deactivate DB

                    CmdService -> CmdService: Cache response in Redis\n(idempotency_key, TTL: 24h)

                    CmdService --> Gateway: 200 OK\n{success: true, message: "Doors locked"}
                    deactivate CmdService
                    Gateway --> App: 200 OK
                    deactivate Gateway
                    App --> User: SUCCESS: Doors locked ✓
                    note over Vehicle, DoorECU #SUCCESS_COLOR
                      Happy Path Complete
                      Total Latency: ~1.2s
                    end note

                else Door Lock Hardware Failure
                    DoorECU --> Vehicle: CAN Response\n0x2A7: ERROR_ACTUATOR_FAILURE
                    deactivate DoorECU

                    Vehicle -> Broker: PUBLISH response\nPayload: {success: false, error: "HARDWARE_FAILURE",\nmessage: "Door lock actuator malfunction"}
                    Broker -> Vehicle: PUBACK
                    Broker --> CmdService: Response message
                    deactivate Vehicle

                    CmdService -> DB: UPDATE commands SET status = 'FAILED',\nerror = 'HARDWARE_FAILURE'\nWHERE id = {uuid}
                    activate DB
                    DB --> CmdService: 1 row updated
                    deactivate DB

                    CmdService -> Monitor: Log hardware failure\n(DTC code, vehicle_id)

                    CmdService --> Gateway: 500 Internal Server Error\n{error: "Hardware failure", code: "HARDWARE_FAILURE"}
                    deactivate CmdService
                    Gateway --> App: 500 Internal Server Error
                    deactivate Gateway
                    App --> User: ERROR: Door lock malfunction.\nPlease visit service center.
                    note over DoorECU #CRITICAL_COLOR
                      FMEA: Door Lock Actuator Failure
                      Severity: 7 (feature unavailable, physical issue)
                      Occurrence: 2 (rare, hardware wear)
                      Detection: 1 (immediate)
                      RPN: 14 (Low-Medium)
                      Mitigation: Log DTC, schedule service
                    end note
                    stop

                else CAN Bus Timeout (>500ms)
                    Vehicle -> Vehicle: CAN timeout detected
                    Vehicle -> Broker: PUBLISH response\nPayload: {success: false, error: "TIMEOUT",\nmessage: "Door ECU not responding"}
                    Broker -> Vehicle: PUBACK
                    Broker --> CmdService: Response message
                    deactivate Vehicle

                    CmdService -> Monitor: Alert: CAN bus timeout\n(vehicle_id, ecu: door_lock)

                    CmdService --> Gateway: 504 Gateway Timeout\n{error: "Vehicle ECU timeout"}
                    deactivate CmdService
                    Gateway --> App: 504 Gateway Timeout
                    deactivate Gateway
                    App --> User: ERROR: Vehicle system timeout.\nTry again.
                    note over Vehicle, DoorECU #WARNING_COLOR
                      FMEA: CAN Bus Communication Failure
                      Severity: 6 (feature unavailable)
                      Occurrence: 3 (network interference, EMI)
                      Detection: 2 (500ms delay)
                      RPN: 36 (Medium)
                      Mitigation: Retry, log diagnostic event
                    end note
                    stop
                end
            end
        end

        ' Timeout handling if no response from vehicle
        CmdService -> CmdService: Wait for response\n(timeout: 25 seconds)

        alt Timeout: No Response from Vehicle
            deactivate Broker
            CmdService -> DB: UPDATE commands SET status = 'TIMEOUT'\nWHERE id = {uuid}
            activate DB
            DB --> CmdService: 1 row updated
            deactivate DB

            CmdService -> Monitor: Log timeout\n(vehicle_id, command_id)

            CmdService --> Gateway: 504 Gateway Timeout\n{error: "Vehicle did not respond", code: "TIMEOUT"}
            deactivate CmdService
            Gateway --> App: 504 Gateway Timeout
            deactivate Gateway
            App --> User: ERROR: Vehicle did not respond.\nTry again.
            note over CmdService, Vehicle #WARNING_COLOR
              FMEA: Vehicle Response Timeout
              Severity: 6 (user expectation not met)
              Occurrence: 4 (network latency, vehicle offline)
              Detection: 8 (25-second delay)
              RPN: 192 (High)
              Mitigation: Reduce timeout to 15s, improve vehicle connectivity monitoring
              Action Item: Add SMS shoulder tap for offline vehicles
            end note
        end
    end
end

== Monitoring and Alerting ==

Monitor -> Monitor: Aggregate metrics (every 30s)\n- Success rate\n- P50/P95/P99 latency\n- Error counts by type

alt Error Rate > 1%
    Monitor -> Monitor: Trigger alert:\n"High error rate for door lock commands"
    Monitor -> Engineer: Slack notification + PagerDuty
else P99 Latency > 5s
    Monitor -> Monitor: Trigger alert:\n"High latency for door lock commands"
    Monitor -> Engineer: Slack notification
else DLQ Messages > 100
    Monitor -> Monitor: Trigger alert:\n"DLQ backlog for door lock commands"
    Monitor -> Engineer: Email + Slack
end

@enduml
```

---

## FMEA Table

| Failure Mode | Severity | Occurrence | Detection | RPN | Mitigation | Status |
|--------------|----------|------------|-----------|-----|------------|--------|
| Authentication failure | 3 | 2 | 1 | 6 | User education, password reset flow | ✅ Acceptable |
| Auth service unavailable | 5 | 1 | 1 | 5 | Auto-scaling, health checks, circuit breaker | ✅ Acceptable |
| Rate limit exceeded | 3 | 2 | 1 | 6 | Client-side throttling, user feedback | ✅ Acceptable |
| Vehicle not provisioned | 4 | 1 | 1 | 4 | Provisioning validation during onboarding | ✅ Acceptable |
| **Vehicle offline** | **6** | **8** | **2** | **96** | SMS shoulder tap (cost concern), last_seen monitoring | ⚠️ **Medium-High** |
| MQTT broker failure | 8 | 1 | 1 | 8 | Multi-region deployment, auto-failover | ✅ Acceptable |
| Invalid vehicle state | 3 | 3 | 1 | 9 | Precondition checks, user education | ✅ Acceptable |
| Door lock actuator failure | 7 | 2 | 1 | 14 | DTC logging, service scheduling | ✅ Acceptable |
| CAN bus timeout | 6 | 3 | 2 | 36 | Retry logic, EMI shielding | ✅ Acceptable |
| **Vehicle response timeout** | **6** | **4** | **8** | **192** | Reduce timeout to 15s, improve connectivity | ❌ **High - Action Required** |
| Database connection pool exhaustion | 9 | 2 | 1 | 18 | Connection pool sizing, monitoring | ✅ Acceptable |
| Duplicate command execution | 2 | 4 | 1 | 8 | Idempotency keys, cache validation | ✅ Acceptable |

**RPN Severity Levels:**
- **Low (RPN < 20):** Acceptable, monitor
- **Medium (RPN 20-100):** Mitigation recommended
- **High (RPN > 100):** Action required, prioritize

---

## Action Items

### High Priority (RPN > 100)

1. **Vehicle Response Timeout (RPN 192)**
   - **Current:** 25-second timeout causes poor user experience
   - **Action:** Reduce timeout to 15 seconds
   - **Additional:** Implement proactive SMS shoulder tap for offline vehicles
   - **Owner:** Backend Team
   - **Deadline:** Q1 2026

### Medium Priority (RPN 20-100)

2. **Vehicle Offline (RPN 96)**
   - **Current:** No automatic wake-up mechanism
   - **Action:** Implement SMS shoulder tap with cost controls ($0.05/message)
   - **Budget:** $5K/month for 100K vehicles (1% offline rate)
   - **Owner:** Product Team (budget approval), Backend Team (implementation)
   - **Deadline:** Q2 2026

3. **CAN Bus Timeout (RPN 36)**
   - **Current:** No retry logic for CAN communication
   - **Action:** Implement 3-retry policy with 200ms backoff
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q2 2026

---

## Test Plan

### Unit Tests

```python
def test_idempotency_duplicate_command():
    """Verify duplicate commands return cached response"""
    key = "test-idempotency-key"
    response1 = execute_lock_command(key)
    response2 = execute_lock_command(key)  # Duplicate
    assert response1 == response2
    assert door_lock_actuations == 1  # Only actuated once
```

### Integration Tests

```python
def test_qos1_duplicate_delivery():
    """Verify QoS 1 duplicate messages handled correctly"""
    publish_command(topic, qos=1, idempotency_key="test-key")

    # Simulate QoS 1 retry (duplicate)
    publish_command(topic, qos=1, idempotency_key="test-key")

    assert vehicle_responses.count == 1  # Single execution
```

### Chaos Tests

```python
def test_mqtt_broker_partition():
    """Verify graceful degradation during broker partition"""
    with broker_partition():
        response = execute_lock_command()
        assert response.status == 503
        assert "Service Unavailable" in response.message

    # Verify DLQ contains failed command
    assert dlq_messages.contains(command_id)
```

---

## Performance Benchmarks

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| P50 Latency | <800ms | 650ms | ✅ Pass |
| P95 Latency | <1.5s | 1.2s | ✅ Pass |
| P99 Latency | <3s | 2.8s | ✅ Pass |
| Success Rate | >99% | 99.3% | ✅ Pass |
| Timeout Rate | <0.5% | 0.8% | ⚠️ Monitor |

---

## Monitoring and Alerts

### Metrics

```
# Prometheus-style metrics
v2c_command_requests_total{command="lock_doors", status="success|failure|timeout"}
v2c_command_latency_seconds{command="lock_doors", percentile="50|95|99"}
v2c_vehicle_state{vehicle_id, state="online|offline"}
v2c_mqtt_publish_errors_total{broker, reason}
v2c_dlq_messages_total{command_type}
```

### Alerts

```yaml
- alert: HighDoorLockFailureRate
  expr: rate(v2c_command_requests_total{command="lock_doors", status="failure"}[5m]) > 0.01
  for: 5m
  severity: P2
  description: "Door lock failure rate {{ $value }}% exceeds 1% threshold"

- alert: MQTTBrokerDown
  expr: up{job="mqtt-broker"} == 0
  for: 1m
  severity: P1
  description: "MQTT broker {{ $labels.instance }} is down"

- alert: HighDLQBacklog
  expr: v2c_dlq_messages_total > 100
  for: 10m
  severity: P2
  description: "DLQ backlog {{ $value }} messages exceeds threshold"
```

---

## References

- [Topic Naming Convention](../standards/TOPIC_NAMING_CONVENTION.md)
- [QoS Selection Guide](../standards/QOS_SELECTION_GUIDE.md)
- [Error Taxonomy (Common.proto)](../../src/main/proto/V2C/Common.proto)
- [ISO 21434 Cybersecurity Engineering](https://www.iso.org/standard/70918.html)
- [MQTT 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

---

## Revision History

| Version | Date       | Author                | Changes                          |
|---------|------------|-----------------------|----------------------------------|
| 1.0     | 2025-10-09 | V2C Architecture Team | Initial FMEA analysis            |

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
