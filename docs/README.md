# Vehicle-to-Cloud Communications Architecture Documentation

## Overview

This repository contains a production-ready MQTT 5.0 communications architecture for connected vehicles, built on AWS IoT Core. The system enables bi-directional communication between vehicles and cloud services with enterprise-grade security, reliability, and scalability.

## Documentation Index

### 📋 API Specifications

- **[AsyncAPI Specification](../asyncapi.yaml)** - Complete MQTT API specification with message schemas, topics, and operations
- **[Protocol Buffer Messages](PROTOCOL_BUFFERS.md)** - Comprehensive documentation of all protobuf message definitions

### 🔧 API Reference

- **[API Reference Guide](api/API_REFERENCE.md)** - Complete API documentation with examples
- **[Documentation Solutions](DOCUMENTATION_SOLUTIONS.md)** - Guide for viewing and working with the various API formats

### 🏗️ Architecture Diagrams

All diagrams are located in [`src/main/doc/puml/`](../src/main/doc/puml/) and rendered as PNG images:

- **[C4 Project Architecture](../src/main/doc/puml/C4_Project_Architecture.png)** - High-level system architecture
- **[MQTT Client Message Lifecycle](../src/main/doc/puml/mqtt_client_message_life_cycle.png)** - Message flow and state transitions
- **[Vehicle Message Header](../src/main/doc/puml/VehicleMessageHeader.png)** - Message header structure
- **[AWS Architecture Example](../src/main/doc/puml/aws_plant_example.png)** - AWS infrastructure deployment

## Quick Start

### View AsyncAPI Specification

The AsyncAPI spec can be viewed in several ways:

1. **AsyncAPI Studio** (Recommended)
   ```bash
   # Open in browser
   open https://studio.asyncapi.com/
   # Import asyncapi.yaml from this repository
   ```

2. **Generate Static HTML**
   ```bash
   npx @asyncapi/generator asyncapi.yaml @asyncapi/html-template -o ./docs/asyncapi-html
   ```

### View Protocol Buffer Documentation

The [PROTOCOL_BUFFERS.md](PROTOCOL_BUFFERS.md) document includes:
- MQTT topic patterns with QoS levels
- Complete message and field tables
- Architecture and sequence diagrams
- Usage examples

## MQTT Topics Overview

| Topic Pattern | Direction | Description |
|--------------|-----------|-------------|
| `v2c/v1/{region}/{vehicle_id}/telemetry/vehicle` | Publish | Real-time vehicle telemetry |
| `v2c/v1/{region}/{vehicle_id}/telemetry/batch` | Publish | Batch telemetry (96% bandwidth savings) |
| `v2c/v1/{region}/{vehicle_id}/command/request` | Subscribe | Remote commands from cloud |
| `v2c/v1/{region}/{vehicle_id}/command/response` | Publish | Command execution status |
| `v2c/v1/{region}/{vehicle_id}/ota/available` | Subscribe | Software update notifications |
| `v2c/v1/{region}/{vehicle_id}/ota/progress` | Publish | OTA download/install progress |
| `v2c/v1/{region}/{vehicle_id}/diagnostics/dtc` | Publish | Diagnostic trouble codes |

## Security

- **Authentication**: Mutual TLS (mTLS) with X.509 certificates
- **Authorization**: AWS IoT Core policies with fine-grained permissions
- **Encryption**: TLS 1.2+ in transit, AES-256 at rest
- **Certificate Rotation**: Automated certificate lifecycle management

## Message Protocols

### Protocol Buffers (protobuf)
All messages are serialized using Protocol Buffers v3 for:
- **Compact size**: 60-70% smaller than JSON
- **Type safety**: Strong typing and validation
- **Forward/backward compatibility**: Extensible schema evolution
- **Performance**: Fast serialization/deserialization

### MQTT 5.0
MQTT 5.0 provides:
- **Quality of Service**: QoS 0, 1, and 2 for reliability
- **Retained Messages**: Last known state persistence
- **User Properties**: Custom metadata per message
- **Request/Response**: Native request-response pattern
- **Topic Aliases**: Bandwidth optimization

## Architecture Highlights

### Key Features
- ✅ **Bi-directional Communication**: Cloud ↔️ Vehicle messaging
- ✅ **Telemetry Batching**: 96% bandwidth reduction
- ✅ **OTA Updates**: Secure firmware/software updates
- ✅ **Remote Commands**: Lock/unlock, climate control, diagnostics
- ✅ **Precision Location**: High-accuracy GPS tracking
- ✅ **Diagnostics**: Real-time DTC reporting and freeze frames

### Performance Characteristics
- **Latency**: < 100ms (P50), < 500ms (P99)
- **Throughput**: 10,000+ vehicles per AWS account
- **Message Size**: 256 bytes (avg), 2 KB (max after compression)
- **Bandwidth**: 1-5 KB/min per vehicle (with batching)

## Development Tools

### Render PlantUML Diagrams
```bash
cd /path/to/axiom-loom-catalog
node scripts/render-plantuml.js vehicle-to-cloud-communications-architecture
```

### Generate Proto Documentation
```bash
node scripts/generate-proto-docs.js vehicle-to-cloud-communications-architecture
```

## Related Documentation

- [AsyncAPI Specification](https://www.asyncapi.com/docs/getting-started) - AsyncAPI documentation
- [Protocol Buffers Guide](https://protobuf.dev/) - Official protobuf documentation
- [MQTT 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) - MQTT protocol reference
- [AWS IoT Core Developer Guide](https://docs.aws.amazon.com/iot/latest/developerguide/) - AWS IoT documentation

## Support

For questions or issues:
1. Check [DOCUMENTATION_SOLUTIONS.md](DOCUMENTATION_SOLUTIONS.md) for viewing options
2. Review [API_REFERENCE.md](api/API_REFERENCE.md) for API details
3. Consult [PROTOCOL_BUFFERS.md](PROTOCOL_BUFFERS.md) for message specifications

---

*Last updated: 2025-10-09*
*Architecture Version: 1.0.0*
*AsyncAPI Version: 3.0.0*
