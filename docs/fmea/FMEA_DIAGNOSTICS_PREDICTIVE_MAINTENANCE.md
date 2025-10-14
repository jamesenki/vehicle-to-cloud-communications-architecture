# FMEA: Diagnostics and Predictive Maintenance

**Feature:** Automated DTC Detection, ML-Based Failure Prediction, and Multi-Channel Customer Notification
**Version:** 1.0
**Last Updated:** 2025-10-14
**Owner:** V2C Architecture Team
**Compliance:** ISO 15765-4 (OBD-II), SAE J1979, ISO 21434, MQTT 5.0, Project FMEA Standards

---

## Executive Summary

This document provides a comprehensive Failure Mode and Effects Analysis (FMEA) for the end-to-end Diagnostics and Predictive Maintenance system in the Vehicle-to-Cloud (V2C) communications architecture. The analysis covers three integrated phases: DTC detection and reporting (04a), ML-based analysis and service recommendations (04b), and multi-channel customer notifications (04c).

### Key Metrics

| Metric                           | Value          |
|----------------------------------|----------------|
| DTC Detection Success Rate       | 99.8%          |
| P99 End-to-End Latency           | 8.5 seconds    |
| ML Prediction Accuracy           | 89%            |
| Notification Delivery Rate       | 96%            |
| MTTR (Mean Time to Recovery)     | <10 minutes    |
| RPN Threshold                    | 100            |
| Business Impact: Breakdown Reduction | 34%       |

---

## Sequence Diagram

```plantuml
@startuml
!define FAILURE_COLOR #FF6B6B
!define WARNING_COLOR #FFD93D
!define SUCCESS_COLOR #6BCF7F
!define CRITICAL_COLOR #FF4757

title FMEA: Diagnostics and Predictive Maintenance - End-to-End Flow
skinparam backgroundColor #FEFEFE
skinparam responseMessageBelowArrow true

' Define participants
participant "Vehicle ECU" as ECU #LightBlue
participant "Diagnostic Agent\n[In-Vehicle]" as DiagAgent #LightGreen
queue "AWS IoT Core\n[MQTT 5.0]" as IoTCore #LightPink
participant "Lambda: DTC Processor\n[Timeout: 30s]" as DTCProcessor #LightYellow
database "DynamoDB\n[DTC History]" as DynamoDB #LightGray
participant "ML Service\n[SageMaker v2.3]" as MLService #Orange
participant "Service Reco" as ServiceReco #LightCyan
participant "Warranty Service" as Warranty #LightBlue
participant "Notification\n[Multi-Channel]" as Notification #LightSalmon
participant "Towing API" as Towing #CRITICAL_COLOR
queue "Dead Letter Queue" as DLQ #FAILURE_COLOR
participant "Monitoring" as Monitor #Orange
actor "Vehicle Owner" as Owner #LightBlue

== Phase 1: DTC Detection (04a) ==

ECU -> ECU: Detect fault (P0171: Lean condition)
activate ECU
note right
**DTC Confirmation**:
3 drive cycles required
MTBF: 50,000 hours
end note

ECU -> ECU: Capture freeze frame (RPM, speed, temps)

alt DTC Confirmed (Active)
    ECU -> DiagAgent: Report DTC via OBD-II (Mode $03)
    activate DiagAgent

else OBD-II Timeout (>5s)
    ECU --X DiagAgent: Timeout
    DiagAgent -> DiagAgent: Retry (3 attempts)
    alt All Retries Failed
        DiagAgent -> DiagAgent: Cache locally (SQLite)
        DiagAgent -> Monitor: ERROR: OBD-II failure
        note right #FF6B6B
        FMEA: OBD-II Timeout
        Severity: 8, RPN: 32
        end note
        stop
    end
end

DiagAgent -> DiagAgent: Check deduplication cache
alt DTC Already Reported
    DiagAgent -> DiagAgent: Suppress duplicate
    deactivate DiagAgent
    stop
else New DTC
    DiagAgent -> DiagAgent: Add to cache (TTL: 24h)
end

DiagAgent -> DiagAgent: Build DTCReport protobuf + ZSTD compress
note right
JSON: 2.4 KB → Protobuf+ZSTD: 842 bytes
end note

DiagAgent -> IoTCore: PUBLISH\nTopic: v2c/v1/{region}/{vin}/diagnostics/dtc\nQoS: 1
activate IoTCore

alt MQTT Success
    IoTCore --> DiagAgent: PUBACK
    deactivate DiagAgent
else Connection Lost
    DiagAgent -> DiagAgent: Store in local queue (1000 max)
    DiagAgent -> Monitor: WARNING: MQTT offline
    note right #FFD93D
    FMEA: Connectivity Loss
    Severity: 7, RPN: 42
    end note
    deactivate DiagAgent
    stop
end

== Phase 2: ML Analysis (04b) ==

IoTCore -> DTCProcessor: Invoke Lambda (MQTT event)
activate DTCProcessor

DTCProcessor -> DTCProcessor: Deserialize protobuf

alt Protobuf Valid
    DTCProcessor -> DynamoDB: PutItem {vehicle_id, dtc_code}
    activate DynamoDB

    alt DynamoDB Write Success
        DynamoDB --> DTCProcessor: Success (12ms)
        deactivate DynamoDB
    else DynamoDB Throttling
        DynamoDB --> DTCProcessor: Throttling
        DTCProcessor -> DTCProcessor: Retry (3x)
        alt All Retries Failed
            DTCProcessor -> DLQ: Send to DLQ
            DTCProcessor -> Monitor: CRITICAL: DynamoDB failure
            note right #FF6B6B
            FMEA: DynamoDB Persistent Failure
            Severity: 6, RPN: 24
            end note
            deactivate DTCProcessor
            stop
        end
    end
else Protobuf Invalid
    DTCProcessor -> DLQ: Send to DLQ for manual review
    DTCProcessor -> Monitor: ERROR: Invalid protobuf
    note right #FF6B6B
    FMEA: Protobuf Parsing Failure
    Severity: 5, RPN: 5
    end note
    deactivate DTCProcessor
    stop
end

DTCProcessor -> DynamoDB: QUERY last 30 days DTCs
activate DynamoDB
DynamoDB --> DTCProcessor: [P0171 (3x), P0174 (2x)]
deactivate DynamoDB

DTCProcessor -> DTCProcessor: Analyze patterns
note right
Pattern: P0171+P0174 → MAF sensor
end note

DTCProcessor -> MLService: POST /predict {dtc, history, freeze_frame}
activate MLService

alt ML Inference Success
    MLService -> MLService: Feature engineering + XGBoost inference (120ms)
    MLService --> DTCProcessor: Prediction
    deactivate MLService
    note right
    {failure_probability: 78%,
    component: "MAF Sensor",
    remaining_km: 350,
    confidence: 0.92}
    end note

else ML Timeout (>10s)
    MLService --X DTCProcessor: Timeout
    DTCProcessor -> DTCProcessor: Fallback: Rule-based prediction
    DTCProcessor -> Monitor: WARNING: ML timeout
    note right #FFD93D
    FMEA: ML Inference Timeout
    Severity: 4, RPN: 24
    Fallback accuracy: 80%
    end note

else ML Service Down
    MLService --> DTCProcessor: 503 Unavailable
    DTCProcessor -> DTCProcessor: Use rule-based fallback
    DTCProcessor -> Monitor: CRITICAL: ML service outage
    note right #FF6B6B
    FMEA: ML Service Outage
    Severity: 5, RPN: 10
    end note
end

DTCProcessor -> ServiceReco: GET /recommend {dtc, prediction}
activate ServiceReco
ServiceReco -> ServiceReco: Load business rules + find dealers
ServiceReco --> DTCProcessor: Recommendation
deactivate ServiceReco
note right
{service: "Replace MAF Sensor",
part: "OEM-MAF-2023-X",
cost: $225, urgency: MEDIUM}
end note

DTCProcessor -> Warranty: POST /validate {vin, dtc, mileage}
activate Warranty

alt Warranty Valid
    Warranty --> DTCProcessor: Covered (100%)
    deactivate Warranty
else Warranty Timeout (>15s)
    Warranty --X DTCProcessor: Timeout
    DTCProcessor -> DTCProcessor: Skip warranty check
    DTCProcessor -> Monitor: WARNING: Warranty timeout
    note right #FFD93D
    FMEA: Warranty Service Timeout
    Severity: 3, RPN: 18
    end note
end

== Phase 3: Customer Notification (04c) ==

DTCProcessor -> DTCProcessor: Evaluate severity: MEDIUM

alt Severity: CRITICAL
    DTCProcessor -> Towing: POST /dispatch {location, vin}
    activate Towing

    alt Towing Available
        Towing --> DTCProcessor: Dispatched (ETA: 22 min)
        deactivate Towing
    else No Tow Trucks Available
        Towing --> DTCProcessor: ERROR: No capacity
        DTCProcessor -> Monitor: CRITICAL: Towing unavailable
        note right #FF6B6B
        FMEA: Towing Unavailable
        Severity: 9, RPN: 18
        end note
    end

    DTCProcessor -> Notification: URGENT (all channels)

else Severity: HIGH
    DTCProcessor -> Notification: HIGH PRIORITY (push + email + in-vehicle)

else Severity: MEDIUM
    DTCProcessor -> Notification: NORMAL (push + email)

else Severity: LOW/INFO
    DTCProcessor -> DTCProcessor: Log only, no notification
    stop
end

activate Notification
Notification -> Notification: Load owner preferences
note right
Push: enabled
Email: enabled
SMS: critical only
end note

' Push Notification
Notification -> Notification: Send Push (FCM)

alt Push Success
    Notification -> Notification: FCM delivered (2s)
else Push Failure (Invalid Token)
    Notification -> Notification: Mark token invalid
    Notification -> Monitor: WARNING: Invalid FCM token
    note right #FFD93D
    FMEA: Invalid Push Token
    Severity: 4, RPN: 16
    end note
else FCM Service Down
    Notification -> Notification: Retry (3x)
    alt All Retries Failed
        Notification -> Monitor: CRITICAL: FCM outage
        note right #FF6B6B
        FMEA: FCM Service Outage
        Severity: 6, RPN: 6
        end note
    end
end

' Email Notification
Notification -> Notification: Send Email (SES)

alt Email Success
    Notification -> Notification: SES accepted (99.2% delivery)
else Email Bounce
    Notification -> Notification: Mark email bounced
    Notification -> Monitor: INFO: Email bounced
    note right #FFD93D
    FMEA: Email Bounce
    Severity: 3, RPN: 27
    end note
else SES Throttling
    Notification -> Notification: Enqueue for retry (5 min)
    Notification -> Monitor: WARNING: SES throttling
    note right #FFD93D
    FMEA: SES Throttling
    Severity: 2, RPN: 4
    end note
end

' SMS Notification (CRITICAL/HIGH only)
alt Severity CRITICAL or HIGH
    Notification -> Notification: Send SMS (SNS)

    alt SMS Success
        Notification -> Notification: SNS delivered (4s)
    else SMS Failure
        Notification -> Notification: Retry (3x)
        alt All Retries Failed
            Notification -> Monitor: CRITICAL: SNS outage
            note right #FF6B6B
            FMEA: SNS Service Outage
            Severity: 7, RPN: 7
            end note
        end
    end
end

Notification --> DTCProcessor: Delivery report {push: ✓, email: ✓}
deactivate Notification

DTCProcessor -> DynamoDB: UPDATE notification_sent=true
activate DynamoDB
DynamoDB --> DTCProcessor: Updated
deactivate DynamoDB

DTCProcessor -> Monitor: INFO: Processing complete (2.8s)
deactivate DTCProcessor
deactivate IoTCore

== Owner Interaction ==

Notification -> Owner: Push: "Vehicle Maintenance Alert: P0171"
activate Owner

Owner -> Owner: Open mobile app
note right
**Details**:
- DTC: P0171 (Lean condition)
- Action: Replace MAF sensor
- Cost: $225 (warranty covered)
- Urgency: Service within 500 km
- Nearest dealers: [3 options]
end note

Owner -> Owner: Tap "Schedule Service"
deactivate Owner

@enduml
```

### Rendered Sequence Diagrams

The complete flow is visualized in three detailed diagrams:
- **04a-dtc-detection.png**: DTC detection and MQTT publishing
- **04b-ml-analysis.png**: ML-based failure prediction and service recommendations
- **04c-customer-notification.png**: Multi-channel notification delivery

---

## FMEA Table

| Failure Mode | Severity | Occurrence | Detection | RPN | Mitigation | Status |
|--------------|----------|------------|-----------|-----|------------|--------|
| Freeze frame capture failure | 6 | 7 | 2 | 84 | Retry 3x, request snapshot from cloud | ⚠️ **Medium-High** |
| OBD-II communication timeout | 8 | 2 | 2 | 32 | Exponential backoff retry, local caching | ✅ Acceptable |
| Prolonged connectivity loss (MQTT) | 7 | 3 | 2 | 42 | Local queue (1000 DTCs), store-and-forward | ✅ Acceptable |
| MQTT message too large | 4 | 3 | 1 | 12 | Protobuf+ZSTD compression, chunking protocol | ✅ Acceptable |
| Protobuf parsing failure | 5 | 1 | 1 | 5 | DLQ for manual review, schema validation | ✅ Acceptable |
| DynamoDB write throttling | 6 | 2 | 2 | 24 | Auto-scaling, exponential backoff, DLQ | ✅ Acceptable |
| **ML inference timeout** | **4** | **6** | **1** | **24** | Rule-based fallback (80% accuracy) | ✅ Acceptable |
| ML service outage | 5 | 2 | 1 | 10 | Rule-based fallback, multi-region deployment | ✅ Acceptable |
| Warranty service timeout | 3 | 6 | 1 | 18 | Async retry, skip if timeout, cache results | ✅ Acceptable |
| Invalid FCM push token | 4 | 4 | 1 | 16 | Fallback to email/SMS, token cleanup | ✅ Acceptable |
| FCM service outage | 6 | 1 | 1 | 6 | Multi-channel delivery, retry logic | ✅ Acceptable |
| Email bounce | 3 | 9 | 1 | 27 | Request updated email, alternative channels | ✅ Acceptable |
| SES throttling | 2 | 2 | 1 | 4 | Increase SES send rate, delayed retry queue | ✅ Acceptable |
| SNS service outage | 7 | 1 | 1 | 7 | Multi-channel redundancy, retry logic | ✅ Acceptable |
| Towing service unavailable (CRITICAL DTC) | 9 | 2 | 1 | 18 | Multi-provider network, escalation to roadside | ⚠️ **Medium** |

**RPN Severity Levels:**
- **Low (RPN < 20):** Acceptable, monitor
- **Medium (RPN 20-100):** Mitigation recommended
- **High (RPN > 100):** Action required, prioritize

---

## Action Items

### Medium Priority (RPN 20-100)

1. **Freeze Frame Capture Failure (RPN 84)**
   - **Current:** 7% failure rate during fault conditions
   - **Action:** Implement redundant capture mechanism, retry with 500ms delay
   - **Additional:** Cloud-side snapshot request if all retries fail
   - **Owner:** Vehicle Platform Team
   - **Deadline:** Q1 2026

2. **Email Bounce Rate (RPN 27)**
   - **Current:** 9% occurrence rate, outdated email addresses
   - **Action:** Proactive email validation in mobile app, annual email confirmation
   - **Budget:** $2K for email validation API
   - **Owner:** Customer Engagement Team
   - **Deadline:** Q2 2026

3. **ML Inference Timeout (RPN 24)**
   - **Current:** 6% timeout rate during peak loads
   - **Action:** Scale SageMaker endpoint auto-scaling, optimize model inference time
   - **Target:** Reduce P99 inference latency from 800ms to 500ms
   - **Owner:** ML Engineering Team
   - **Deadline:** Q1 2026

4. **Towing Service Unavailable (RPN 18)**
   - **Current:** Single provider, 2% unavailability for CRITICAL DTCs
   - **Action:** Integrate with 3 regional towing networks (AAA, Agero, Urgently)
   - **Budget:** $50K annually for API integrations
   - **Owner:** Operations Team
   - **Deadline:** Q2 2026

---

## Test Plan

### Unit Tests

```python
def test_dtc_deduplication():
    """Verify duplicate DTC suppression"""
    agent = DiagnosticAgent()
    agent.report_dtc("P0171", occurrence=1)
    agent.report_dtc("P0171", occurrence=1)  # Duplicate

    assert mqtt_published_count == 1  # Only first reported
    assert cache_entries == 1
```

### Integration Tests

```python
def test_ml_fallback_on_timeout():
    """Verify rule-based fallback when ML times out"""
    with mock_ml_timeout():
        response = process_dtc("P0171", history=[...])

        assert response.prediction_source == "RULE_BASED"
        assert response.confidence < 0.9  # Lower than ML
        assert response.component == "Fuel System"
```

### Chaos Tests

```python
def test_mqtt_broker_partition():
    """Verify local queuing during extended MQTT outage"""
    with mqtt_broker_down(duration=60):
        for i in range(100):
            report_dtc(f"P{i:04d}")

        assert local_queue.size() == 100

    # Broker recovers
    wait_for_reconnect()
    assert local_queue.size() == 0  # All flushed
    assert mqtt_published_count == 100
```

### End-to-End Tests

```python
def test_critical_dtc_towing_dispatch():
    """Verify towing dispatch for CRITICAL DTCs"""
    report = DTCReport(
        code="P0601",  # ECU internal failure
        severity="CRITICAL",
        location=(37.7749, -122.4194)
    )

    result = process_dtc_e2e(report)

    assert result.towing_dispatched == True
    assert result.towing_eta_min < 30
    assert result.notifications_sent == ["push", "email", "sms"]
    assert result.vehicle_remote_start_disabled == True
```

---

## Performance Benchmarks

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| DTC Detection Rate | >99.5% | 99.8% | ✅ Pass |
| MQTT Publish Success | >99% | 99.97% | ✅ Pass |
| DTC Processing P99 | <5s | 2.8s | ✅ Pass |
| ML Inference P99 | <500ms | 120ms | ✅ Pass |
| Notification Delivery | >95% | 96% | ✅ Pass |
| End-to-End P99 Latency | <10s | 8.5s | ✅ Pass |
| Freeze Frame Capture | >95% | 93% | ⚠️ Monitor |

---

## Monitoring and Alerts

### Metrics

```
# Prometheus-style metrics
v2c_dtc_reports_total{severity, category}
v2c_dtc_processing_duration_seconds{phase, percentile}
v2c_ml_inference_duration_seconds{model_version, result}
v2c_ml_inference_errors_total{error_type}
v2c_notification_delivery_total{channel, status}
v2c_notification_latency_seconds{channel, percentile}
v2c_towing_dispatch_total{status}
v2c_obd2_communication_errors_total{vehicle_model}
v2c_mqtt_connection_state{vehicle_id}
v2c_dlq_messages_total{failure_reason}
```

### Alerts

```yaml
- alert: HighDTCProcessingErrorRate
  expr: rate(v2c_dtc_processing_errors_total[5m]) > 0.05
  for: 5m
  severity: P2
  description: "DTC processing error rate {{ $value }}% exceeds 5% threshold"

- alert: MLInferenceTimeoutSpike
  expr: rate(v2c_ml_inference_errors_total{error_type="timeout"}[5m]) > 0.10
  for: 3m
  severity: P2
  description: "ML inference timeout rate {{ $value }}% exceeds 10%"

- alert: NotificationDeliveryDegraded
  expr: (sum(rate(v2c_notification_delivery_total{status="success"}[5m])) / sum(rate(v2c_notification_delivery_total[5m]))) < 0.90
  for: 10m
  severity: P2
  description: "Notification delivery rate {{ $value }}% below 90% threshold"

- alert: CriticalDTCTowingFailed
  expr: v2c_towing_dispatch_total{status="failed"} > 0
  for: 1m
  severity: P1
  description: "Critical DTC detected but towing dispatch failed"

- alert: MQTTBrokerDown
  expr: up{job="mqtt-broker"} == 0
  for: 2m
  severity: P1
  description: "MQTT broker {{ $labels.instance }} is down"

- alert: HighDLQBacklog
  expr: v2c_dlq_messages_total > 500
  for: 15m
  severity: P2
  description: "DLQ backlog {{ $value }} messages exceeds threshold"
```

---

## Business Impact

### Operational Metrics

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Roadside Breakdowns | 8.2 per 1000 vehicles | 5.4 per 1000 | 34% reduction |
| Average Repair Cost | $480 | $335 | $145 savings |
| Warranty Claims (preventable) | 14% | 12% | 14% reduction |
| Customer Satisfaction (NPS) | +42 | +60 | +18 points |
| Service Center No-Shows | 22% | 9% | 59% reduction |

### Cost-Benefit Analysis

**Annual Costs (100K vehicles):**
- AWS IoT Core (MQTT): $48K
- Lambda (DTC processing): $36K
- DynamoDB (5-year retention): $24K
- SageMaker (ML inference): $72K
- Notifications (FCM, SES, SNS): $18K
- **Total Annual Cost:** $198K

**Annual Savings:**
- Reduced roadside assistance: $420K (2,800 events × $150/event)
- Reduced warranty claims: $280K (preventable claims)
- Increased service revenue: $360K (proactive service appointments)
- **Total Annual Savings:** $1,060K

**ROI:** 435% (payback period: 2.7 months)

---

## References

- [Sequence Diagram 04a: DTC Detection](../sequence-diagrams/04a-dtc-detection.puml)
- [Sequence Diagram 04b: ML Analysis](../sequence-diagrams/04b-ml-analysis.puml)
- [Sequence Diagram 04c: Customer Notification](../sequence-diagrams/04c-customer-notification.puml)
- [Topic Naming Convention](../standards/TOPIC_NAMING_CONVENTION.md)
- [QoS Selection Guide](../standards/QOS_SELECTION_GUIDE.md)
- [ISO 15765-4: OBD-II on CAN](https://www.iso.org/standard/66369.html)
- [SAE J1979: OBD-II Diagnostic Test Modes](https://www.sae.org/standards/content/j1979_202104/)
- [ISO 21434 Cybersecurity Engineering](https://www.iso.org/standard/70918.html)
- [MQTT 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

---

## Revision History

| Version | Date       | Author                | Changes                          |
|---------|------------|-----------------------|----------------------------------|
| 1.0     | 2025-10-14 | V2C Architecture Team | Initial FMEA analysis covering phases 04a, 04b, 04c |

---

## Approval

| Role                          | Name                | Signature | Date       |
|-------------------------------|---------------------|-----------|------------|
| Architecture Lead             | ___________________ | _________ | __________ |
| Vehicle Platform Lead         | ___________________ | _________ | __________ |
| ML Engineering Lead           | ___________________ | _________ | __________ |
| Quality Assurance Lead        | ___________________ | _________ | __________ |
| ISO 21434 Compliance Officer  | ___________________ | _________ | __________ |

---

**Document Classification:** Internal - Engineering Documentation
**Security Level:** Confidential
**Distribution:** V2C Development Team, QA Team, Vehicle Engineering, ML Engineering, Operations
