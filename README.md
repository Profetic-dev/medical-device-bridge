# Medical Device Protocol Bridge

A reverse-engineered integration layer for proprietary medical devices, plus a full-featured clinic management platform.

Built by [PROFETIC](https://profetic.dev)

---

## Overview

Legacy medical devices often ship with outdated, limited software. This project reverse-engineers proprietary serial protocols and builds a modern, extensible platform on top.

```
Medical Devices → Protocol Bridge → REST API → React Clinic Platform
     ↓                  ↓               ↓              ↓
  3 device types    Auto-detect    Real-time     Full patient
  reverse-engineered   & route      status       management
```

**Result:** Devices that "couldn't talk to modern systems" now integrate with a modern web-based clinic platform.

---

## Architecture

### 1. Protocol Reverse Engineering

Captured thousands of serial operations via USB sniffing, then built parsers to reconstruct the undocumented protocol.

#### Sniffed Protocol Analysis

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
            
            # Extract timing data for protocol reconstruction
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
    'R': 4,     # Run command: 4 char payload  
    'P': 4,     # Program command: 4 char payload
    'W': None,  # Write data: variable until 'V' terminator
    'i': None,  # Flash write: variable until DEL (0x7F)
}

def fix_missing_bytes(operations):
    """
    Reconstruct dropped bytes using protocol context.
    Timing anomalies (31-32ms where 16ms expected) indicate missing cycles.
    """
    current_command = None
    payload_remaining = 0
    
    for i, op in enumerate(operations):
        # Track command state to know expected payload lengths
        if op['action'] == 'WRITE' and chr(op['value']) in COMMAND_PAYLOADS:
            current_command = chr(op['value'])
            
        # Detect timing anomalies indicating dropped bytes
        if op['action'] == 'READ' and op.get('delay_before_ms') in [31, 32]:
            write_op = find_previous_write(operations, i)
            if write_op and write_op.get('value') != op.get('value'):
                # Echo mismatch — reconstruct missing cycle
                insert_missing_cycle(fixed_ops, write_op, op)
```

### 2. Unified Device Bridge

Auto-detects device type and routes to the appropriate protocol implementation:

```python
def detect_device(handle):
    """
    Run Stage 1 init and detect device type from response signature.
    Returns: ('MR72', serial) or ('EM272B', serial) or (None, None)
    """
    # Execute init sequence (identical for all devices)
    for op in STAGE1_INIT_OPS:
        execute_init_operation(handle, op)
    
    # Device type indicated by response to Y command
    # K = MR72/MagRez, H = EM272B, E = EM27, etc.
    device_identifiers = {
        ord('K'): 'MR72',
        ord('H'): 'EM272B', 
        ord('E'): 'EM27',
        ord('G'): 'MR772',
    }
    
    for _ in range(15):
        b = read_byte(handle)
        if b in device_identifiers:
            serial = read_serial_number(handle)
            return device_identifiers[b], serial
    
    return None, None
```

### 3. REST API Layer

Flask API enabling web-based control:

```python
@app.route('/api/send-to-device', methods=['POST'])
def send_to_device():
    """Store treatment on device (auto-detects device type)"""
    
    # Auto-detect device
    device_type, serial = detect_device(handle)
    
    # Route to appropriate protocol
    if device_type == 'MR72':
        operations = mr72_swap_treatment(MR72_PROTOCOL, name, reps, rows)
    elif device_type == 'EM272B':
        operations = em272b_swap_treatment(EM272B_PROTOCOL, treatment)
    
    # Execute protocol stages
    execute_stage(handle, 1, operations)  # Init
    execute_stage(handle, 2, operations)  # Store treatment
    
    return jsonify({
        'success': True,
        'device_type': device_type,
        'serial_number': serial
    })

@app.route('/api/device/start', methods=['POST'])
def start_device():
    """Start treatment with optional resume from specific row"""
    start_row = request.json.get('start_row', 0)
    
    # Threaded playback with real-time status updates
    playback_thread = threading.Thread(
        target=run_playback_thread, 
        args=(False, start_row)
    )
    playback_thread.start()
```

### 4. React Clinic Platform

Full-featured web application for clinical operations:

```jsx
const TreatmentCompiler = ({ selectedFiles, onRemoveFile }) => {
  const [deviceStatus, setDeviceStatus] = useState({
    connected: false,
    is_running: false,
    is_loaded: false
  });
  
  // Drag-and-drop treatment ordering
  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor)
  );
  
  // Real-time device status polling
  useEffect(() => {
    const interval = setInterval(async () => {
      const res = await fetch('http://localhost:8000/api/device/status');
      const status = await res.json();
      setDeviceStatus(status);
    }, 1000);
    return () => clearInterval(interval);
  }, []);
  
  // Queue change detection — auto-unload when modified
  useEffect(() => {
    if (deviceStatus.is_loaded && queueChanged) {
      fetch('http://localhost:8000/api/device/unload', { method: 'POST' });
    }
  }, [sortedFiles]);
```

**Platform Features:**
- Treatment file browser and editor
- Drag-and-drop treatment compilation
- Patient record management
- Session history and analytics
- Multi-device support from single interface
- Real-time playback status with pause/resume

### 5. Analytics API

Backend service for clinic data management:

```python
@app.route('/api/patients', methods=['GET', 'POST'])
def patients():
    """Patient CRUD with treatment history"""
    
@app.route('/api/sessions', methods=['POST'])
def record_session():
    """Log treatment session with device data"""
    
@app.route('/api/analytics/frequency-usage')
def frequency_analytics():
    """Aggregate frequency usage across all sessions"""
```

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      React Frontend (Vite)                          │
│   File Browser │ Treatment Editor │ Compiler │ Patient Manager      │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│    Bridge Server (:8000) │    │   Analytics API (:5000)  │
│    Flask + Windows API   │    │    Flask + MySQL         │
│                          │    │                          │
│  • Device auto-detect    │    │  • User authentication   │
│  • Protocol execution    │    │  • Patient records       │
│  • Playback control      │    │  • Session logging       │
│  • Real-time status      │    │  • Usage analytics       │
└──────────────────────────┘    └──────────────────────────┘
              │                               │
              ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│   Device Hardware        │    │      MySQL Database      │
│   (MR72/EM272B/etc)      │    │   (resetmfg_analytics)   │
└──────────────────────────┘    └──────────────────────────┘
```

---

## Tech Stack

- **Protocol Layer:** Python, ctypes (Windows API), PySerial
- **Bridge Server:** Flask, threading
- **Frontend:** React 18, Vite, Ant Design, dnd-kit
- **Analytics API:** Flask, PyMySQL, SQLAlchemy
- **Database:** MySQL

---

## Results

- **3** proprietary device types reverse-engineered
- **100%** replacement of limited vendor software
- **Full clinic platform** — not just device control
- **Pause/resume** playback with row-level precision
- **Auto-detection** — plug in any supported device

---

## Private Repository

This is a showcase repository demonstrating architecture and capabilities.

Full implementation available for client engagements.

**Contact:** [hello@profetic.dev](mailto:hello@profetic.dev)

---

© 2025 PROFETIC
