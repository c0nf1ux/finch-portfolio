# Finch BLE: Accessibility & Usability

## Command-Line Interface (CLI) Accessibility

### Text-Based Interface
- No GUI dependency — accessible to users with visual impairments via screen readers
- Clear, descriptive command names: `move()`, `turn()`, `led()` (not cryptic `m()`, `t()`)
- Status messages include text descriptions alongside numeric values
- Error messages provide actionable next steps, not just error codes

### Cross-Platform Support
- Python CLI works on Windows, macOS, Linux
- Bleak library automatically detects compatible Bluetooth adapters
- No platform-specific barriers — same code runs everywhere

## Documentation for Diverse Learning Styles

### Code Examples
- Concrete examples accompany each API function
- Comments explain the "why" (strategy/intent), not just the "what"
- Behavioral code (Phase 1-4) annotates state machines with human-readable transitions

### Terminology
- Plain language explanations (e.g., "distance in centimeters" vs. cryptic unit labels)
- Glossary for technical terms: BLE, UUID, characteristic, MTU, RSSI
- Links to authoritative sources (pxt-finch, BirdBrain, micro:bit) for deeper learning

## Testing & Validation

### Comprehensive Test Coverage
- 100+ tests ensure behaviors work as documented
- Test names match user intent: `test_move_forward_10cm()` not `test_motor_1()`
- Edge cases documented: max distance (500cm), acceleration limits (5-30 cm/s)

### Real Hardware Validation
- All behaviors tested on actual Finch robots (not simulated)
- Room exploration tests verify distances, wall avoidance, prey behavior
- Interactive testing with live animals (cats) ensures naturalistic behavior

## Inclusive Design Philosophy

### Low Barrier to Entry
- No MakeCode IDE installation required
- Python 3.9+ and `pip install` is all that's needed
- Bleak automatically finds compatible Bluetooth adapters
- Works with standard Windows/macOS/Linux equipment

### Cognitive Load Management
- API surface is ~19 functions (not overwhelming)
- Behaviors are composable (users combine simple moves into complex strategies)
- Documented protocols mean no guessing about byte formats or timing

## What This Demonstrates

1. **Accessibility as Requirement** — Text-based interface benefits not just screen-reader users, but anyone without GUI access (embedded systems, remote access, etc.)
2. **Documentation for Diverse Learners** — Code examples + plain language serve visual learners, readers, and hands-on experimenters
3. **Validation on Real Hardware** — Testing with actual Finch robots and live animals proves the implementation works in messy reality, not just in ideal conditions

---

For additional compliance and quality standards, see PORTFOLIO-COMPLIANCE.md.
