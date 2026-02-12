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

#### Sniffed Protocol Analysis

Captured thousands of serial operations via USB sniffing, then built parsers to reconstruct the protocol:

```python
def parse_protocol_file(filepath):
    """
    Parse sniffed protocol output into structured operations.
    Handles consolidated operations, timing data, and multiple encodings.
    """
    operations = []
    current_stage = None
    
    # Handle different file encodings from various capture tools
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            lines = f.readlines()
    except UnicodeDecodeError:
        with open(filepath, 'r', encoding='utf-16') as f:
            lines = f.readlines()
    
    for line in lines:
        # Parse operation lines: [XXXX] OPERATION details [+XXms]
        op_match = re.match(r'\[(\d+)(?:-(\d+))?\]\s+(.+)', line)
        if op_match:
            op_start = int(op_match.group(1))
            op_end = int(op_match.group(2)) if op_match.group(2) else op_start
            op_content = op_match.group(3)
            
            # Extract timing data
            delay_match = re.search(r'\[(\+)?(\d+)ms\]', op_content)
            delay_ms = int(delay_match.group(2)) if delay_match else 0
            
            # Route to type-specific parsers...
```

#### Context-Aware Missing Byte Recovery

The USB sniffer occasionally drops fast back-to-back writes. We detect and fix these using protocol context:

```python
# Command context tracking for payload-aware fixes
COMMAND_PAYLOADS = {
    'I': 54,    # Treatment name: exactly 54 chars
    'R': 4,     # Run command: 4 char payload ("00hh")  
    'P': 4,     # Program command: 4 char payload
    'W': None,  # Write data: variable until 'V' terminator
    'i': None,  # Flash write: variable until DEL (0x7F)
}

def fix_missing_bytes(operations):
    """
    Reconstruct dropped bytes using protocol context.
    
    When sniffer drops a WRITE, the subsequent READ echo won't match.
    We detect mismatches and reconstruct the missing cycle.
    """
    # Track command state
    current_command = None
    payload_remaining = 0
    
    for i, op in enumerate(operations):
        if op['action'] == 'WRITE':
            byte_val = op.get('value', 0)
            
            # Check if inside a data payload first
            if payload_remaining > 0:
                payload_remaining -= 1
                continue
                
            # Detect new command
            if chr(byte_val) in COMMAND_PAYLOADS:
                current_command = chr(byte_val)
                # Payload starts after echo...
```

#### Timing-Based Anomaly Detection

Protocol timing reveals sniffer drops — a 31-32ms delay where 16ms is expected indicates a missing byte cycle:

```python
# FIX: Mismatched echo with anomalous delay
if (op['action'] == 'READ' and 
        op.get('delay_before_ms') in [31, 32] and  # Double normal delay
        i >= 2):
    
    # Find the WRITE this should be echoing
    write_op = find_previous_write(operations, i)
    
    if write_op and write_op.get('value') != op.get('value'):
        # Echo doesn't match — sniffer dropped a full cycle
        # Reconstruct: READ(correct) → PURGE → WRITE(dropped) → SET_TIMEOUTS → READ
        insert_missing_cycle(fixed_ops, write_op, op)
```

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
