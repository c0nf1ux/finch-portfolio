# Finch BLE: Technical Architecture & Protocol Design

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│ APPLICATION LAYER                                               │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ PreySimulatorPhase5C (Orchestrator)                      │   │
│ │ - Manages all components lifecycle                       │   │
│ │ - Integrates async tasks with sync behavior engine      │   │
│ │ - Logs state transitions, events, metrics               │   │
│ └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│ BEHAVIOR LAYER                                                  │
│ ┌──────────────┐  ┌─────────────┐  ┌────────────────────────┐ │
│ │ Hierarchical │  │ Threat Model│  │ Engagement Tracking    │ │
│ │ FSM          │  │ (Velocity)  │  │ (5-min sliding window) │ │
│ │              │  │             │  │ - Multipliers          │ │
│ │ FORAGE       │  │ threat_level│  │ - Vocalization freq    │ │
│ │ ├─WANDER     │  │ = velocity /│  │ - Movement aggressiv   │ │
│ │ ├─EAT        │  │   distance  │  │ - Recovery urgency     │ │
│ │ FREEZE       │  │             │  └────────────────────────┘ │
│ │ ├─WATCHFUL   │  │ Predict     │                              │
│ │ ├─ALERT      │  │ before react│  ┌────────────────────────┐ │
│ │ RECOVER      │  │             │  │ Boid Movement          │ │
│ │ ├─RESTING    │  │             │  │ - Seek behavior        │ │
│ │ ├─CAUTIOUS   │  │             │  │ - Flee behavior        │ │
│ │ FLEE (term)  │  └─────────────┘  │ - Wander behavior      │ │
│ └──────────────┘                   │ - Wall avoidance       │ │
│                                    │ - Velocity scaling     │ │
│                                    └────────────────────────┘ │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ Spatial Memory Grid                                      │   │
│ │ - 10×10 arena divided into cells                        │   │
│ │ - Per cell: visit timestamp, threat encounter count    │   │
│ │ - Enables stale area detection + unsafe zone avoidance │   │
│ │ - Heatmap generation for analysis                       │   │
│ └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│ COORDINATION LAYER (Async Background Tasks)                    │
│ ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐│
│ │ SensorReader   │  │ MotorController│  │ Behavior Engine    ││
│ │ (Background)   │  │ (Background)   │  │ (Synchronous)      ││
│ │                │  │                │  │                    ││
│ │ Runs every     │  │ Consumes motor │  │ Executes state     ││
│ │ 100ms          │  │ commands from  │  │ transitions,       ││
│ │                │  │ queue          │  │ generates motor    ││
│ │ Reads all      │  │                │  │ commands           ││
│ │ sensors        │  │ Handles timing │  │                    ││
│ │                │  │ (Phase 5b:     │  │ Maintains threat   ││
│ │ Applies 3-pt   │  │ predictive     │  │ model, engagement  ││
│ │ moving avg     │  │ execution)     │  │ tracking, spatial  ││
│ │                │  │                │  │ memory             ││
│ │ Writes to      │  │ Logs [EXEC]    │  │                    ││
│ │ queue          │  │ messages       │  │ Logs [BEHAVIOR]    ││
│ │                │  │                │  │ events             ││
│ │ Logs raw data  │  │ Catches        │  │                    ││
│ │               │  │ exceptions     │  │ Outputs JSON       ││
│ └────────────────┘  └────────────────┘  │ metrics            ││
│                                         └────────────────────┘│
│ ┌─────────────────────────────────────────────────────────┐   │
│ │ asyncio.Queue (Inter-Component Communication)          │   │
│ │ - sensor_queue: SensorReading events (100ms rate)      │   │
│ │ - motor_queue: MotorCommand events (on-demand)         │   │
│ └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│ HARDWARE INTERFACE LAYER                                        │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ Finch.py (High-Level BLE API)                           │   │
│ │                                                          │   │
│ │ Async methods:                                          │   │
│ │ - connect() / disconnect() — BLE lifecycle              │   │
│ │ - read_sensors() → SensorReading — Poll all sensors    │   │
│ │ - move_forward(cm) / move_backward(cm) — Distance ctrl │   │
│ │ - turn(degrees) — In-place rotation                     │   │
│ │ - set_beak(r, g, b) — RGB LED control                  │   │
│ │ - buzz(frequency, duration) — Buzzer output            │   │
│ │ - set_motors(left_speed, right_speed) — Raw motor ctrl │   │
│ │                                                          │   │
│ │ Internal:                                               │   │
│ │ - BleakClient for async BLE communication              │   │
│ │ - Notification callback for RX (sensor frames)         │   │
│ │ - TX UUID for motor/LED/buzzer commands                │   │
│ │ - Frame parsing (20-byte telemetry)                    │   │
│ └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│ COMMUNICATION PROTOCOL (BLE UART)                               │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ Service: Nordic UART (micro:bit v2 standard)            │   │
│ │ TX UUID: 6e400002-b5a3-f393-e0a9-e50e24dcca9e          │   │
│ │ RX UUID: 6e400003-b5a3-f393-e0a9-e50e24dcca9e          │   │
│ │                                                          │   │
│ │ RX Frame (20 bytes, sent by robot every 100ms):        │   │
│ │ ┌──────────────────────────────────────────┐           │   │
│ │ │ Byte 0-1:   Distance (cm, big-endian)    │           │   │
│ │ │ Byte 2:     Light LEFT (0-255)           │           │   │
│ │ │ Byte 3:     Light RIGHT (0-255)          │           │   │
│ │ │ Byte 4:     Line LEFT (7-bit) + motor ctrl│          │   │
│ │ │ Byte 5:     Line RIGHT (7-bit)           │           │   │
│ │ │ Byte 6:     Battery voltage              │           │   │
│ │ │ Byte 7-9:   Encoder LEFT (24-bit signed) │           │   │
│ │ │ Byte 10-12: Encoder RIGHT (24-bit signed)│           │   │
│ │ │ Byte 13-19: Padding / IMU data (TBD)     │           │   │
│ │ └──────────────────────────────────────────┘           │   │
│ │                                                          │   │
│ │ TX Commands (variable-length):                          │   │
│ │ Motor: [0x4D, left_speed, right_speed]                 │   │
│ │ LED:   [0x4C, r, g, b, duration_ms]                    │   │
│ │ Buzzer:[0x42, frequency_hz, duration_ms]               │   │
│ └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│ ROBOT HARDWARE                                                  │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ BBC micro:bit v2 (Finch 2.0)                            │   │
│ │                                                          │   │
│ │ Motors: Two independent DC motors with encoders        │   │
│ │ Sensors:                                                │   │
│ │ - Ultrasonic distance (0-400cm)                         │   │
│ │ - Light sensors (2x, 0-255 each)                       │   │
│ │ - Line sensors (2x, 0-127 reflectivity)                │   │
│ │ - Encoders (2x, 24-bit signed ticks)                   │   │
│ │ - Battery voltage (ADC value 0-255)                    │   │
│ │ - IMU (9-axis, future use)                             │   │
│ │                                                          │   │
│ │ Outputs:                                                │   │
│ │ - RGB Beak LED (24-bit color)                          │   │
│ │ - Buzzer (frequency + duration)                        │   │
│ │ - Tail wiggle (differential motor speed)               │   │
│ │ - Motor movement (encoder-based distance/angle)        │   │
│ └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Structures

### SensorReading (events.py)
```python
@dataclass
class SensorReading:
    timestamp: datetime
    distance_cm: float              # Raw ultrasonic (0-400cm)
    distance_filtered: float        # 3-point moving average
    light_left: int                 # 0-255 (0=dark, 255=bright)
    light_right: int                # 0-255
    battery: int                    # 0-255 (ADC value)
    # Derived fields calculated in threat model:
    velocity_estimate: float        # cm/s (distance change rate)
    threat_level: float             # 0.0-1.0 (0=safe, 1.0=critical)
```

### MotorCommand (events.py)
```python
@dataclass
class MotorCommand:
    cmd_type: MotorCommandType      # MOVE_FORWARD, MOVE_BACKWARD, TURN, SET_MOTORS
    distance_cm: float              # For MOVE_* commands
    angle_deg: float                # For TURN command
    speed_pct: int                  # 0-100% motor speed
    left_speed: int                 # -100 to 100 (for SET_MOTORS)
    right_speed: int                # -100 to 100
    execution_time_ms: int          # 0 = immediate, >0 = absolute Unix timestamp (Phase 5b)
```

### BehaviorEvent (events.py)
```python
@dataclass
class BehaviorEvent:
    timestamp: datetime
    event_type: str                 # "state_transition", "vocalization", "movement", "motor_command"
    details: dict                   # Flexible structure for event-specific data
    # Example details for state transition:
    # {"from_state": "FORAGE", "to_state": "FREEZE", "trigger": "distance"}
    # Example details for vocalization:
    # {"frequency_hz": 1200, "duration_ms": 500, "type": "alarm_chirp"}
```

---

## Protocol Details

### 20-Byte RX Telemetry Frame

**Conversion Factors:**
```python
# Distance (bytes 0-1)
distance_cm = (frame[0] << 8 | frame[1]) * 0.091

# Light sensors (bytes 2-3)
light_left = frame[2]   # Already 0-255
light_right = frame[3]

# Line sensors (bytes 4-5, 7-bit values)
line_left = 100 - (((frame[4] & 0x7F) - 6) * 100 / 121)
line_right = 100 - (((frame[5] & 0x7F) - 6) * 100 / 121)

# Motor control flag (byte 4, bit 7)
motor_control_active = bool((frame[4] & 0x80) >> 7)

# Battery (byte 6)
battery_level = frame[6]  # ADC value 0-255

# Encoders (bytes 7-12, 24-bit signed integers)
enc_left = (frame[7] << 16) | (frame[8] << 8) | frame[9]
if enc_left >= 0x800000:
    enc_left -= 0x1000000  # Convert to signed

enc_right = (frame[10] << 16) | (frame[11] << 8) | frame[12]
if enc_right >= 0x800000:
    enc_right -= 0x1000000

# Motor correction (hardware-specific)
# RIGHT motor is 5.7% slower than LEFT
# Applied as 1.06x speed boost during reverse/FLIGHT state
```

### Motor Command TX Format
```
Command Type: Motor Speed
Format: [0x4D, left_speed, right_speed]
- 0x4D = motor command opcode
- left_speed: -100 to 100 (-100 = max reverse, 0 = stop, 100 = max forward)
- right_speed: -100 to 100

Example: [0x4D, 75, 79]  // Move forward-right at 75-79% speed
         (Use 79 for right to compensate for 5.7% motor drift)
```

### LED Command TX Format
```
Command Type: Beak LED
Format: [0x4C, r, g, b, duration_lo, duration_hi]
- 0x4C = LED command opcode
- r, g, b: 0-255 (RGB color)
- duration: 16-bit little-endian milliseconds

Example: [0x4C, 255, 0, 0, 232, 3]  // Red beak for 1000ms (232 + 256*3 = 1000)
```

### Buzzer Command TX Format
```
Command Type: Buzzer
Format: [0x42, freq_lo, freq_hi, duration_lo, duration_hi]
- 0x42 = buzzer command opcode
- freq: 16-bit frequency in Hz (little-endian)
- duration: 16-bit milliseconds (little-endian)

Example: [0x42, 208, 4, 232, 1]  // 1200Hz for 488ms (208 + 256*4 = 1200Hz)
```

---

## Conversion Factors & Calibration

### Motor to Distance
```python
TICKS_PER_ROTATION = 792
CM_PER_ROTATION = 49.7
CM_PER_TICK = CM_PER_ROTATION / TICKS_PER_ROTATION  # 0.0627 cm/tick

# To move 10cm forward:
# target_ticks = 10 / 0.0627 = 159.5 ≈ 160 ticks
```

### Motor to Angle
```python
WHEEL_DISTANCE_CM = 10.0  # Distance between left/right wheels
TICKS_PER_DEGREE = 4.335  # Empirically measured

# To turn 90 degrees:
# target_ticks = 90 * 4.335 = 390 ticks per motor (opposite directions)
```

### Light Sensor LED Correction
```python
# Beak LED affects light sensor readings
# Correction polynomial applied in finch.py
def apply_led_correction(raw_light, beak_r, beak_g, beak_b):
    # From BirdBrain official documentation
    # Scales light readings to account for LED bleed
    return raw_light * correction_factor(beak_color)
```

### Motor Drift Compensation
```python
# RIGHT motor observed 5.7% slower than LEFT
# Correction factor: 1.06x speed boost
# Applied specifically during FLIGHT state (reverse movement)

# Example:
# Desired: reverse at 80% speed
# LEFT:  80% speed
# RIGHT: 80% * 1.06 = 84.8% speed (rounded to 85%)
# Result: Straight reverse line
```

---

## Async Communication Model

### Queue-Based Architecture
```
┌─────────────────────┐
│ SensorReader Task   │
│ (runs every 100ms)  │
└──────────┬──────────┘
           │ puts SensorReading
           ▼
    ┌─────────────────┐
    │ sensor_queue    │
    │ (asyncio.Queue) │
    └─────────────────┘
           │ gets
           ▼
    ┌──────────────────┐
    │ BehaviorEngine   │
    │ (sync, called    │
    │  from orchestr.) │
    └──────────┬───────┘
               │ enqueues MotorCommand
               ▼
        ┌─────────────────┐
        │ motor_queue     │
        │ (asyncio.Queue) │
        └─────────────────┘
               │ gets
               ▼
    ┌──────────────────────┐
    │ MotorController Task │
    │ (runs continuously)  │
    └──────────┬───────────┘
               │ sends to Finch BLE API
               ▼
          ┌──────────┐
          │ Finch    │
          │ Robot    │
          └──────────┘
```

### Key Properties
- **Non-blocking:** SensorReader and MotorController don't wait for each other
- **Rate-independent:** BehaviorEngine runs at variable rate (triggered by sensor updates)
- **Scalable:** Can add multiple robots with separate queues/tasks
- **Fault-tolerant:** One component failure doesn't block others (separate try/except)

---

## State Machine Design

### Hierarchical FSM (Phase 5c)

**Main States:**
```
FORAGE (distance > 150cm)
├─ WANDER: Explore arena, random patrol
├─ EAT: Pause and forage (standing still)
└─ [auto-transitions based on engagement]

FREEZE (80cm < distance ≤ 150cm)
├─ WATCHFUL: Stillness, attention high
├─ ALERT: Maximum vigilance, preparation to flee
└─ [auto-transitions based on threat level]

RECOVER (post-FLEE, re-assessing)
├─ RESTING: Regain composure, lower threat
├─ CAUTIOUS: Gradual rearm of FLIGHT threshold
└─ [auto-transitions after 30s timeout]

FLEE (distance < 80cm)
└─ [terminal state—immediate escape to safety]
```

**Engagement Multipliers:**
```
Engagement Score = threat_encounters_5min / (events_total_5min + 1)
Range: 0.0 (safe) to 1.0 (high hunting pressure)

Effects:
- Movement aggressiveness: 0.5x (safe) to 1.5x (hunted)
- Vocalization frequency: 0.2x to 2.0x
- Exploration bias: 0.8 (explore when safe) to 0.2 (evade when hunted)
- Recovery urgency: 10s (relaxed) to 2s (high alert)
```

---

## Performance Characteristics

### Latency Budget
```
Sensor Read → Behavior Decision → Motor Execute:
├─ Sensor read: ~100ms (BLE polling interval)
├─ Behavior decision: ~50ms (FSM transition, threat calc)
├─ Motor command TX: <50ms (BLE write)
└─ Motor response: ~50ms (firmware execution)

Total end-to-end latency: ~250ms
(Acceptable for prey simulation; faster than biological reaction time)
```

### Throughput
```
Sensor Readings: 10/sec (100ms interval)
Motor Commands: ~1000/sec theoretical
  (limited by robot firmware, not Python code)
Behavior Events: ~50/min (depends on threat stimulus)
```

### Memory Usage
```
SensorReader: ~2MB (deque, event objects)
MotorController: <1MB (queue, command objects)
BehaviorEngine: ~10MB (FSM state, spatial grid, threat model history)
Finch API: ~5MB (BLE client, notification buffers)
Total: ~18MB (acceptable for embedded use)
```

---

## Error Handling & Robustness

### BLE Connection Failures
```python
# Connection retry logic
max_retries = 5
for attempt in range(max_retries):
    try:
        async with BleakClient(device_address) as client:
            # Connected, proceed
    except BleakDeviceNotFoundError:
        # Device not in range, retry in 2s
        await asyncio.sleep(2)
    except BleakError:
        # BLE error, retry
        await asyncio.sleep(1)
    finally:
        # Cleanup resources
```

### Sensor Reading Errors
```python
# Validate frame size
if len(frame) != 20:
    raise ValueError(f"Invalid frame size: {len(frame)}, expected 20")

# Clamp distance to max
distance_sanitized = min(raw_reading.distance_cm, 400)

# Filter out NaN/inf values
if math.isnan(distance_filtered) or math.isinf(distance_filtered):
    distance_filtered = last_known_distance
```

### Motor Command Timeout
```python
# MotorController waits for command with 1s timeout
motor_cmd = await asyncio.wait_for(
    self.motor_queue.get(),
    timeout=1.0
)
# If timeout, loop continues (allows stop-and-wait behavior)
```

---

## Scaling Considerations

### Single Robot (Current)
- One Finch robot controlled by one Python process
- Uses one asyncio event loop
- ~18MB memory, <5% CPU

### Multi-Robot (Future)
```python
# Hypothetical 10-robot scenario
tasks = []
for robot_address in robot_addresses:
    finch = Finch(robot_address)
    await finch.connect()
    
    sensor_queue = asyncio.Queue()
    motor_queue = asyncio.Queue()
    
    sensor_reader = SensorReader(finch, sensor_queue)
    motor_controller = MotorController(finch, motor_queue)
    behavior_engine = PreySimulatorPhase5C(...)
    
    # All run concurrently on same event loop
    tasks.extend([
        sensor_reader.start(),
        motor_controller.start(),
        behavior_engine.run()
    ])

await asyncio.gather(*tasks)
```

**Estimated capacity:** 50-100 robots per process (limited by BLE radio range and event loop throughput)

---

## Summary

This architecture demonstrates:
- ✅ **Async-first design** for non-blocking I/O
- ✅ **Layered separation of concerns** (protocol → API → behavior → orchestration)
- ✅ **Hardware-aware compensation** (motor drift, sensor correction)
- ✅ **Production-grade error handling** (retries, timeouts, validation)
- ✅ **Scalable to multi-robot** (async tasks, queue-based communication)
- ✅ **Testable design** (mockable components, data-driven tests)

**Status:** Ready for deployment, education, or further development.
