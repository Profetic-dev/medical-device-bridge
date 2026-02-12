# Medical Device Protocol Bridge

A reverse-engineered integration layer for proprietary medical devices, plus a full clinic management platform.

Built by [PROFETIC](https://profetic.dev)

---

## Overview

Legacy medical devices often ship with outdated, limited software. This project reverse-engineers proprietary serial protocols and builds a modern, extensible platform on top.

```
Medical Devices → Protocol Bridge → Unified API → Clinic Platform
```

**Result:** Devices that "couldn't talk to modern systems" now integrate seamlessly with custom workflows.

---

## Architecture

### 1. Protocol Reverse Engineering

Analyzed undocumented serial communication to decode:
- Command structures and opcodes
- Timing requirements and handshakes
- Response parsing and error states
- Device-specific quirks and edge cases

**Devices supported:** 3 PEMF therapy device types with automatic detection.

### 2. Device Abstraction Layer

Unified interface regardless of underlying hardware:

```python
# Same API across all device types
class DeviceBridge:
    def connect(self, port: str) -> bool
    def get_status(self) -> DeviceStatus
    def start_session(self, params: SessionConfig) -> Session
    def stop_session(self) -> None
    
# Auto-detection handles device differences
bridge = DeviceBridge.auto_detect()  # Returns correct implementation
```

**Features:**
- Automatic device type detection on connection
- Plug-and-play switching between devices
- Connection health monitoring and auto-reconnect
- Graceful error handling

### 3. Clinic Management Platform

Full-featured web application for clinical operations:

**Patient Management**
- Patient records and history
- Treatment plans and protocols
- Session scheduling

**Session Tracking**
- Real-time device monitoring
- Session logging with parameters
- Treatment history and notes

**Analytics Dashboard**
- Usage statistics
- Treatment outcomes tracking
- Device utilization reports

### 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    React Frontend                            │
│         (Patient UI / Session Control / Analytics)           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Flask REST API                          │
│            (Auth / Patients / Sessions / Reports)            │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │  MySQL   │   │  Bridge  │   │   Device     │
        │ Database │   │  Layer   │   │  Hardware    │
        └──────────┘   └──────────┘   └──────────────┘
```

---

## Tech Stack

- **Bridge:** Python, PySerial
- **API:** Flask, SQLAlchemy
- **Frontend:** React
- **Database:** MySQL
- **Deployment:** Production-ready

---

## Results

- **3** device types reverse-engineered and integrated
- **100%** replacement of limited vendor software
- **Zero** dependency on original manufacturer
- **Extensible** architecture for future device types

---

## Private Repository

This is a showcase repository demonstrating architecture and capabilities.

Full implementation available for client engagements.

**Contact:** [hello@profetic.dev](mailto:hello@profetic.dev)

---

© 2025 PROFETIC
