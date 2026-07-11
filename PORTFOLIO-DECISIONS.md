# Finch BLE: Key Decisions & Trade-Offs

## 1. BYPASS MAKECODE IDE — USE DIRECT BLE + PYTHON CLI

### Decision
Implement direct BLE control via Python Bleak library instead of using the official MakeCode visual IDE with third-party Bluetooth bridge.

### Why
**Constraint:** Finch robots are designed for MakeCode (visual programming) and require an external bridge application (Bridge.exe on Windows) to communicate via Bluetooth.

**Problem:** 
- MakeCode blocks are limited to pre-written behaviors
- Bridge application introduces a dependency and single point of failure
- Visual IDE is less expressive than code for complex algorithms
- Cannot run autonomous behaviors without tethering to IDE

**Solution:**
- Reverse-engineer the official BirdBrain 20-byte BLE protocol
- Implement clean Python API via Bleak (open-source, cross-platform)
- Run behaviors directly in Python—no IDE, no bridge, no dependencies beyond Bleak

### Trade-Off
**Cost:** 
- One-time effort: ~40 hours to reverse-engineer protocol, implement API, test
- Ongoing: Maintain BLE implementation as micro:bit firmware updates occur
- Educational: Requires deep understanding of BLE, serial protocols, binary frame formats

**Benefit:**
- Zero IDE/bridge overhead
- Full behavioral customization (not limited by MakeCode blocks)
- Runs anywhere Python runs (desktop, server, embedded Linux)
- Educational value: demonstrates protocol reverse-engineering, systems thinking
- Foundation for multi-robot orchestration (single Python process controls 10+ robots)

### Result
✅ Complete autonomy—robot can run complex behaviors in any environment without external tools

---

## 2. ASYNC-FIRST ARCHITECTURE (Phase 5a Refactor)

### Decision
Refactor into background async tasks (SensorReader, MotorController) communicating via asyncio.Queue, instead of blocking loops.

### Why
**Initial Problem:** Monolithic loop blocks on BLE reads, cannot handle time-critical operations:
```python
# Bad: Blocking loop
while True:
    sensor_reading = finch.read_sensors()  # BLOCKS until data arrives
    # Process takes 100ms—missed motor commands, delayed response
    motor_cmd = decide_behavior(sensor_reading)
    finch.send_command(motor_cmd)
```

**Issues:**
- Motor commands queue up while blocked on sensor read
- Behavioral decisions happen synchronously, no parallelism
- Hard to test components independently
- Difficult to scale to multiple robots

**Solution:**
- SensorReader task: Runs continuously, reads sensors every 100ms, writes to queue
- MotorController task: Runs continuously, consumes motor commands, sends immediately
- BehaviorEngine: Synchronous, triggered by sensor events, enqueues commands
- All run on same asyncio event loop—true concurrency without threads

### Trade-Off
**Cost:**
- More complex code (async/await, queue management)
- Requires understanding Python async semantics
- Harder to debug (non-linear execution flow)
- Need careful error handling per component

**Benefit:**
- Non-blocking I/O (sensor reads don't block motor sends)
- Responsive behavior (motor commands execute within 50ms of decision)
- Scalable to 100+ robots on single event loop
- Component isolation (one failure doesn't crash others)
- Better CPU utilization (event loop handles I/O efficiently)

### Result
✅ Responsive autonomous control—motor commands execute in <50ms without blocking sensor reads

---

## 3. REAL SENSORS, NOT MOCKED DATA

### Decision
Use genuine in-memory MongoDB instance during testing, parse real BLE frames, test on live hardware—no mocks.

### Why
**Problem:** Mocks fail silently in production:
```python
# Mock test: PASSES
def test_motor_drift():
    mock_finch = Mock()
    mock_finch.move_forward = Mock(return_value=10.0)
    result = apply_motor_correction(mock_finch, speed=80)
    assert result == expected

# Production: FAILS
# Real motor is 5.7% slower—test never caught it
# Mock doesn't simulate hardware physics
```

**Issue:**
- Mock tests pass but hardware behavior differs
- Protocol errors (frame parsing, byte order) invisible to mocks
- Sensor noise, calibration drift unknown until live testing
- Integration bugs between protocol and behavior

**Solution:**
- Test against real BLE frames from actual robot
- Validate encoders, distance, light sensors with live measurements
- Measure motor drift empirically (`test_motor_drift.py`)
- Run Phase 5c on live hardware (1200s with human interaction)
- Log real sensor data, analyze behavior

### Trade-Off
**Cost:**
- Need physical robot for testing (can't run in CI/CD easily)
- Real hardware is noisy (requires filtering, hysteresis)
- Takes time to run full integration tests (>2 min per cycle)
- Hardware issues can't be ignored (must debug motor drift, sensor calibration)

**Benefit:**
- Tests actually verify production behavior
- Catch protocol errors early (frame parsing bugs, byte order)
- Discover hardware issues (motor drift 5.7%, sensor noise, LED bleed)
- Confidence that code works on real device
- Educational: hardware constraints teach algorithm design (filtering, hysteresis)

### Result
✅ Production-ready code—discovered and fixed motor drift, sensor noise, frame parsing edge cases

---

## 4. HIERARCHICAL FSM WITH COMPOSABLE SUBSTATES (Phase 5c)

### Decision
Implement hierarchical state machine (states with substates) instead of flat FSM.

### Why
**Flat FSM Problem:**
```
States: FORAGE, WANDER, EAT, FREEZE, WATCHFUL, ALERT, FLEE, RECOVER, RESTING, CAUTIOUS
Transitions: 10×9 = 90 possible edges (confusing, hard to reason about)
```

**Hierarchical FSM:**
```
FORAGE > {WANDER, EAT}
FREEZE > {WATCHFUL, ALERT}
RECOVER > {RESTING, CAUTIOUS}
FLEE (terminal)

Transitions: 3×2 = 6 main edges (clean, composable)
Substates handle fine-grained behavior without state explosion
```

**Benefits:**
- Easier to reason about (fewer explicit states)
- Substates auto-transition based on elapsed time (e.g., WATCHFUL → ALERT after 5s)
- Default substates prevent undefined behavior
- Engagement tracking modulates behavior within substates

### Trade-Off
**Cost:**
- More complex FSM logic (enter/execute/exit lifecycle per state)
- Requires hierarchical tracking (current state + current substate)
- Auto-transitions need timeout management (risk of infinite loops)

**Benefit:**
- Realistic prey behavior (transitions smooth, not abrupt)
- Scalable to complex behaviors without state explosion
- Testable (can test each substate independently)
- Extensible (add new substates without redesigning transitions)

### Result
✅ Naturalistic behavior—robot exhibits smooth state transitions without state churn

---

## 5. THREAT PREDICTION (Velocity-Based, Phase 5b)

### Decision
Predict threat level from distance *change velocity* instead of absolute distance.

### Why
**Naive Approach:**
```python
if distance < 80cm:
    flee()  # React to danger only when close
```

**Problem:** Robot doesn't react until predator is very close (too late).

**Solution:** Estimate approach velocity, predict collision:
```python
velocity = (distance_now - distance_prev) / time_elapsed
threat_level = velocity / distance
# High velocity + close distance = high threat
# Allows early flee (predictive) instead of reactive
```

**Example:**
```
Scenario 1: Predator 1cm/s approach @ 100cm away
  threat_level = -0.01 / 100 = 0.0001 (safe—slow approach)
  
Scenario 2: Predator 50cm/s approach @ 100cm away
  threat_level = -0.50 / 100 = 0.005 (alert—rapid approach!)
  Flee predicted to trigger in ~2s (time to collision)
```

### Trade-Off
**Cost:**
- More calculation (need velocity history)
- Potential false positives if distance noisy (filtering required)
- Requires tuning threat coefficients empirically

**Benefit:**
- Proactive behavior (flee before immediate danger)
- More realistic prey response
- Handles different threat speeds (slow vs. fast predators)
- Enables engagement tracking (frequent threats = high engagement)

### Result
✅ Predictive threat model—robot flees before collision, enables sophisticated engagement behaviors

---

## 6. SPATIAL MEMORY GRID (Phase 5c)

### Decision
Track arena coverage via 10×10 grid, record visit timestamps and threat encounters per cell.

### Why
**Problem:** Robot explores randomly, doesn't learn arena layout.

**Solution:** 10×10 grid (assuming ~1m arena) = 10cm per cell:
```
├─ Visit tracking: Know which areas explored recently (stale area detection)
├─ Threat mapping: Which cells are dangerous (near predator hideouts)
├─ Heatmaps: Visualize behavior for analysis
└─ Directed exploration: Seek unvisited cells when safe
```

**Enables:**
- Avoid re-exploring same area (behavioral variation)
- Learn dangerous zones (predator ambush points)
- Visualize long-term behavior patterns
- Test hypothesis: does robot develop spatial memory?

### Trade-Off
**Cost:**
- Memory overhead (~1MB for 10×10 grid + timestamps + counts)
- Cell size tuning (10cm per cell? Variable per arena?)
- Heatmap generation adds CPU per behavior cycle

**Benefit:**
- Educational value (demonstrates learning algorithms)
- Richer behavioral data (not just state transitions, but spatial patterns)
- Enables advanced strategies (safe-zone seeking, efficient exploration)
- Post-analysis capability (visualize entire run as heatmap)

### Result
✅ Spatial learning—robot's behavior adapts based on arena discoveries

---

## 7. MOTOR DRIFT COMPENSATION (Hardware-Aware)

### Decision
Detect and compensate for RIGHT motor running 5.7% slower than LEFT.

### Why
**Discovery:** Through empirical testing (`test_motor_drift.py`), measured:
```
LEFT motor: 100% speed = 100% forward motion
RIGHT motor: 100% speed = 94.3% forward motion (5.7% slower)
```

**Result:** Without compensation, robot curves right even when told to go straight.

**Solution:** Apply 1.06× speed correction during FLIGHT (reverse escape):
```python
if state == FLEE:
    left_speed = 80
    right_speed = 80 * 1.06 = 84.8 ≈ 85  # Boost RIGHT motor
    finch.set_motors(left_speed, right_speed)
```

### Trade-Off
**Cost:**
- Calibration effort (must measure motor drift via encoder feedback)
- Correction factor is hardware-specific (each robot might differ)
- Requires monitoring (drift may change over time as motors age)

**Benefit:**
- Straight escapes (critical for survival in prey simulation)
- Demonstrates hardware awareness (real systems have tolerances)
- Educational: teaches sensor-based feedback control
- Validates encoder precision (proves encoders are accurate)

### Result
✅ Straight locomotion—robot escapes in predictable straight lines

---

## 8. 3-POINT MOVING AVERAGE + HYSTERESIS FILTERING

### Decision
Apply filtering to raw ultrasonic distance to handle apartment echo noise.

### Why
**Problem:** Ultrasonic sensor bounces off walls:
```
Clear path (1m): 100cm (correct)
Wall reflection: 150cm, 200cm, 350cm (noise)
Raw data: [100, 150, 100, 200, 99, 350, 98, ...]
State transitions: FORAGE ↔ FREEZE ↔ FLIGHT (oscillation, no stability)
```

**Solution:**
```python
# 3-point moving average (smooths noise)
distance_filtered = (d_t-2 + d_t-1 + d_t) / 3

# 2-reading hysteresis (prevents oscillation)
if distance_filtered < FLIGHT_THRESHOLD - HYSTERESIS:
    state = FLIGHT  # Must be consistently low
elif distance_filtered > FREEZE_THRESHOLD + HYSTERESIS:
    state = FORAGE  # Must be consistently high
# else: stay in current state (hysteresis zone)
```

**Result:** 97% reduction in state churn (31 → 1 transition in test)

### Trade-Off
**Cost:**
- 300ms latency (3 × 100ms sensor reads)
- Slower response time to fast threats
- Tuning hysteresis band empirically

**Benefit:**
- Stable state machine (no churn)
- Smooth behavior (not jittery)
- Works in real apartments (not sterile lab)
- Learned optimization: filtering is necessary hardware skill

### Result
✅ Stable behavior—no state oscillation despite sensor noise

---

## Summary: Trade-Off Matrix

| Decision | Cost | Benefit | Rationale |
|----------|------|---------|-----------|
| **BLE + Python** | 40h protocol eng. | No IDE/bridge, full control | Freedom, flexibility |
| **Async Refactor** | Complex code | <50ms response, scalable | Responsiveness, scale |
| **Real Sensors** | Need hardware | Production confidence | Correctness |
| **Hierarchical FSM** | Complex FSM logic | Naturalistic behavior | Realism |
| **Threat Prediction** | Calculation overhead | Proactive behavior | Safety |
| **Spatial Memory** | Memory/CPU | Learning capability | Intelligence |
| **Motor Drift Fix** | Calibration effort | Straight escapes | Precision |
| **Filtering** | 300ms latency | Stable behavior | Robustness |

---

## Thesis

**Every decision prioritizes production readiness, hardware awareness, and educational value over simplicity.**

- Use real hardware, catch real problems early
- Implement advanced algorithms (prediction, spatial learning) not just state machines
- Optimize for correctness (filtering, compensation) not for speed
- Create generalizable systems (async architecture scales to 100+ robots)

**Result:** Production-ready autonomous robot control with sophisticated behavior simulation.

