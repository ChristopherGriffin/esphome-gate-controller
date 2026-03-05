# Gate Controller — QA Test Procedure

Pre-publish functional verification covering every feature of the gate controller. Work through the sections in order — the sequence minimises unnecessary gate trips and builds on state established in prior sections.

---

## Before You Start

| Requirement | Notes |
|-------------|-------|
| Firmware flashed to ESP32 | |
| `secrets.yaml` populated | WiFi, OTA password, API key |
| Device connected to Home Assistant | API active |
| `input_number.entry_gate_duration` created in HA | min: 1, max: 300, step: 1, unit: s |
| Gate wired and physically operational | H-bridge, endstops, motor connected |
| ESPHome web server accessible | `http://<device-ip>` for log inspection |

**Reset state:** Hold the panel Reboot/Reset button for 10+ seconds to clear any previously learned travel times and reboot clean. Manually position the gate at the **closed endstop** before starting.

---

## 1 — Initial State Detection on Boot

The controller checks both endstops on boot and sets the FSM to the correct starting state automatically.

### Scenario 1.1 — Boot at closed endstop

Boot the device with the gate physically at the closed endstop.

| Check | Expected | Pass |
|-------|----------|------|
| Close endstop binary sensor | ON | [ ] |
| FSM state (within 2s of boot) | CLOSED | [ ] |
| Open LED / Close LED | Off / Solid | [ ] |
| Cover position | 0% | [ ] |

### Scenario 1.2 — Boot at open endstop

Manually position the gate at the open endstop, then reboot.

| Check | Expected | Pass |
|-------|----------|------|
| Open endstop binary sensor | ON | [ ] |
| FSM state (within 2s of boot) | OPEN | [ ] |
| Open LED / Close LED | Solid / Off | [ ] |
| Cover position | 100% | [ ] |

### Scenario 1.3 — Boot mid-travel (position unknown)

Manually position the gate mid-travel so neither endstop is active, then reboot.

| Check | Expected | Pass |
|-------|----------|------|
| Both endstop sensors | OFF | [ ] |
| FSM state | STOPPED_UNKNOWN | [ ] |
| Open LED / Close LED | Alternating fast flash (opposite phase) | [ ] |

---

## 2 — Timing Learning (First Run)

The controller measures full travel time on the first run and stores it in flash. This calibration is required before position control (Section 7) and soft stop (Section 8) are meaningful. Both directions are learned in this combined run.

> During a learning run, soft start is intentionally disabled so speed is constant and the timing is accurate.

### Scenario 2.1 — Learn open time

Precondition: Learned times cleared (see Before You Start). State = CLOSED.

Send OPEN from the HA cover entity.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | OPENING | [ ] |
| Log message | "Timing not learned: using default open time" | [ ] |
| Log message | "TIMING RUN ARMED" | [ ] |
| Soft Start Active sensor | Stays OFF for entire run | [ ] |
| Motor current | Steady from start (no soft ramp) | [ ] |
| Open LED / Close LED | Fast flash / Off | [ ] |

Gate travels to open endstop and stops.

| Check | Expected | Pass |
|-------|----------|------|
| Log message | "ENDSTOP: OPEN — emergency motor kill" | [ ] |
| FSM state | OPEN | [ ] |
| Learned Open Time sensor | Non-zero value (measured seconds) | [ ] |
| Gate Position sensor | ~100% | [ ] |
| Open LED / Close LED | Solid / Off | [ ] |

### Scenario 2.2 — Learn close time

Precondition: State = OPEN (from 2.1).

Send CLOSE from HA.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | CLOSING | [ ] |
| Log message | "TIMING RUN ARMED" | [ ] |
| Soft Start Active sensor | Stays OFF for entire run | [ ] |

Gate travels to close endstop and stops.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | CLOSED | [ ] |
| Learned Close Time sensor | Non-zero value | [ ] |
| Open LED / Close LED | Off / Solid | [ ] |

### Scenario 2.3 — Learned times survive reboot

Note the learned open and close times, then reboot the device.

| Check | Expected | Pass |
|-------|----------|------|
| Learned Open Time after reboot | Same as before | [ ] |
| Learned Close Time after reboot | Same as before | [ ] |
| "Timing not learned" warning on next motion | Does NOT appear | [ ] |

---

## 3 — Basic Open / Close / Direction Reversal

Precondition: Timing learned. Gate at closed endstop. State = CLOSED.

### Scenario 3.1 — Open via HA with soft start

Send OPEN from the HA cover entity.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | OPENING | [ ] |
| Soft Start Active sensor | ON at start, transitions OFF mid-travel | [ ] |
| Log message | "Ramp complete at X% travel" | [ ] |
| FSM state at open endstop | OPEN (no countdown) | [ ] |
| Gate Position sensor | ~100% | [ ] |

### Scenario 3.2 — Close via HA

Precondition: State = OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after CLOSE command | CLOSING | [ ] |
| Open LED / Close LED | Off / Fast flash | [ ] |
| FSM state at close endstop | CLOSED | [ ] |
| Gate Position sensor | ~0% | [ ] |

### Scenario 3.3 — Direction reversal mid-travel (CLOSE while OPENING)

Send OPEN, wait until gate is 30–50% open, then send CLOSE.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after CLOSE command | CLOSING immediately (no pause) | [ ] |
| Gate reverses without stopping | Yes | [ ] |
| Gate reaches close endstop | State = CLOSED | [ ] |

### Scenario 3.4 — Direction reversal mid-travel (OPEN while CLOSING)

Send CLOSE, wait until 30–50% closed, then send OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after OPEN command | OPENING immediately | [ ] |
| Gate reaches open endstop | State = OPEN | [ ] |

---

## 4 — Cycle Button & Auto-Close

### Scenario 4.1 — Full auto-close cycle

Precondition: State = CLOSED. Set `input_number.entry_gate_duration` = 10 seconds.

Press Cycle (panel button or HA button entity).

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | OPENING_CYCLE | [ ] |
| Open LED / Close LED | Fast flash / Off | [ ] |

Gate reaches open endstop.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | OPEN_CYCLE | [ ] |
| Open LED / Close LED | Slow pulse (1 Hz) / Off | [ ] |
| Close Countdown sensor | Counting down from 10 | [ ] |
| Auto Close Armed sensor | ON | [ ] |

Countdown expires.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | CLOSING | [ ] |
| Gate closes to endstop | State = CLOSED | [ ] |
| Close Countdown sensor | Resets to 10 | [ ] |
| Auto Close Armed sensor | OFF | [ ] |

### Scenario 4.2 — Cancel auto-close with Cycle button

Precondition: State = OPEN_CYCLE, countdown active.

| Check after pressing Cycle | Expected | Pass |
|---------------------------|----------|------|
| FSM state | OPEN (countdown cancelled) | [ ] |
| Open LED | Solid | [ ] |
| Auto Close Armed sensor | OFF | [ ] |

### Scenario 4.3 — Cancel auto-close with OPEN command

Precondition: State = OPEN_CYCLE, countdown active.

| Check after sending OPEN | Expected | Pass |
|--------------------------|----------|------|
| FSM state | OPEN | [ ] |
| Countdown | Stopped | [ ] |

### Scenario 4.4 — CLOSE command during auto-close countdown

Precondition: State = OPEN_CYCLE, countdown active.

| Check after sending CLOSE | Expected | Pass |
|---------------------------|----------|------|
| FSM state | CLOSING immediately | [ ] |
| Auto Close Armed sensor | OFF | [ ] |

### Scenario 4.5 — CYCLE from OPEN closes directly (no auto-close)

Precondition: State = OPEN (arrived via OPEN command, no countdown armed).

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after Cycle | CLOSING | [ ] |
| Gate closes to endstop | State = CLOSED | [ ] |

### Scenario 4.6 — Countdown pauses when gate leaves open endstop

Precondition: State = OPEN_CYCLE, countdown active (set to 30s for this test).

Note the current countdown value, then briefly jog the gate away from the open endstop.

| Check | Expected | Pass |
|-------|----------|------|
| Countdown while off endstop | Stops decrementing | [ ] |
| Countdown after returning to endstop | Resumes from where it paused | [ ] |

### Scenario 4.7 — Pause mid-OPENING_CYCLE with Cycle, then resume with Cycle

Precondition: State = CLOSED.

Press Cycle, wait until mid-travel, press Cycle again.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after second Cycle | PAUSED_FROM_OPENING | [ ] |
| Gate | Stopped | [ ] |
| Both LEDs | Fast flash | [ ] |

Press Cycle a third time.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | CLOSING | [ ] |
| Gate closes to endstop | State = CLOSED | [ ] |

### Scenario 4.8 — Pause mid-CLOSING, resume with Cycle arms auto-close

Precondition: State = OPEN.

Send CLOSE, wait until mid-travel, press Cycle.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after Cycle | PAUSED_FROM_CLOSING | [ ] |
| Both LEDs | Fast flash | [ ] |

Press Cycle again.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | OPENING_CYCLE | [ ] |
| Gate reaches open endstop | State = OPEN_CYCLE | [ ] |
| Countdown starts | Yes | [ ] |

---

## 5 — Hold-to-Operate (Jog)

Jog is a dead man's switch — the gate moves only while the button is held. It bypasses the FSM entirely; the FSM state does not change during a jog.

### Scenario 5.1 — Jog open, verify FSM unchanged

Precondition: State = CLOSED.

| Check while holding OPEN jog button | Expected | Pass |
|--------------------------------------|----------|------|
| Gate movement | Moving open | [ ] |
| Open LED / Close LED | Fast flash / Off | [ ] |
| FSM state | Still CLOSED | [ ] |
| Motor Current sensor | Positive value | [ ] |

| Check after releasing OPEN jog button | Expected | Pass |
|----------------------------------------|----------|------|
| Gate | Stops immediately | [ ] |
| FSM state | Still CLOSED | [ ] |

### Scenario 5.2 — Jog close, verify FSM unchanged

Precondition: State = OPEN.

| Check while holding CLOSE jog button | Expected | Pass |
|---------------------------------------|----------|------|
| Gate movement | Moving close | [ ] |
| Open LED / Close LED | Off / Fast flash | [ ] |
| FSM state | Still OPEN | [ ] |

| Check after releasing | Expected | Pass |
|-----------------------|----------|------|
| Gate | Stops immediately | [ ] |
| FSM state | Still OPEN | [ ] |

### Scenario 5.3 — Jog stops at endstop

Position the gate within a few seconds of the open endstop, then hold the OPEN jog button.

| Check | Expected | Pass |
|-------|----------|------|
| Gate at/approaching open endstop | Motor stops (endstop lockout) | [ ] |
| FSM state | Unchanged | [ ] |

### Scenario 5.4 — Jog does not disrupt auto-close countdown

Precondition: State = OPEN_CYCLE, countdown active.

Briefly hold CLOSE jog button (gate moves off endstop), then release.

| Check | Expected | Pass |
|-------|----------|------|
| Countdown while off endstop | Paused (not decrementing) | [ ] |
| FSM state after releasing jog | Still OPEN_CYCLE | [ ] |
| Countdown after gate returns to endstop | Resumes | [ ] |

---

## 6 — Stop & Pause Mid-Travel

### Scenario 6.1 — Stop button during opening

Send OPEN, wait until mid-travel, then press the Stop button.

| Check | Expected | Pass |
|-------|----------|------|
| Gate | Stops immediately | [ ] |
| FSM state | STOPPED | [ ] |
| Both LEDs | Fast flash | [ ] |
| Gate resumes on OPEN command | Yes, State = OPENING | [ ] |

### Scenario 6.2 — Stop button during closing

Send CLOSE, wait until mid-travel, then press Stop.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state | STOPPED | [ ] |
| Gate resumes on CLOSE command | Yes, State = CLOSING | [ ] |

### Scenario 6.3 — Stop from HA cover entity

During any motion, click Stop on the HA cover entity.

| Check | Expected | Pass |
|-------|----------|------|
| Gate | Stops immediately | [ ] |
| FSM state | STOPPED | [ ] |

### Scenario 6.4 — Stop has no effect from a resting state

Precondition: State = CLOSED.

| Check after pressing Stop | Expected | Pass |
|---------------------------|----------|------|
| Gate | No movement | [ ] |
| FSM state | Still CLOSED | [ ] |

### Scenario 6.5 — Resume from PAUSED_FROM_OPENING

Precondition: State = PAUSED_FROM_OPENING (paused while opening).

| Check after OPEN command | Expected | Pass |
|--------------------------|----------|------|
| FSM state | OPENING | [ ] |
| Gate | Continues toward open endstop | [ ] |

| Check after CLOSE command (from PAUSED_FROM_OPENING) | Expected | Pass |
|------------------------------------------------------|----------|------|
| FSM state | CLOSING | [ ] |

### Scenario 6.6 — Resume from PAUSED_FROM_CLOSING

Precondition: State = PAUSED_FROM_CLOSING (paused while closing).

| Check after CLOSE command | Expected | Pass |
|---------------------------|----------|------|
| FSM state | CLOSING | [ ] |
| Gate | Continues toward close endstop | [ ] |

| Check after OPEN command (from PAUSED_FROM_CLOSING) | Expected | Pass |
|-----------------------------------------------------|----------|------|
| FSM state | OPENING | [ ] |

---

## 7 — Position Control

Precondition: Timing is learned (Section 2 complete).

### Scenario 7.1 — Move to 50% from closed

Precondition: State = CLOSED. Set the HA cover slider to 50%.

| Check | Expected | Pass |
|-------|----------|------|
| FSM state during travel | OPENING | [ ] |
| Gate Position sensor | Increases during travel | [ ] |
| Soft Stop Active sensor | ON as gate approaches target | [ ] |
| Gate stops at | ~50% (±5% acceptable) | [ ] |
| FSM state after stop | STOPPED | [ ] |

### Scenario 7.2 — Move to 75% from 50%, then 25% from 75%

| Check | Expected | Pass |
|-------|----------|------|
| Slider to 75% from STOPPED at 50% | Gate opens to ~75% and stops | [ ] |
| Slider to 25% from STOPPED at 75% | Gate closes to ~25% and stops | [ ] |

### Scenario 7.3 — Position command cancels auto-close countdown

Precondition: State = OPEN_CYCLE, countdown active.

| Check after setting slider to 50% | Expected | Pass |
|-----------------------------------|----------|------|
| Auto Close Armed sensor | OFF | [ ] |
| FSM state | CLOSING | [ ] |
| Gate stops at | ~50% | [ ] |

---

## 8 — Soft Start & Soft Stop

### Scenario 8.1 — Soft start ramp in normal operation

Precondition: Timing learned. State = CLOSED. Send OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| Soft Start Active sensor at motion start | ON | [ ] |
| Motor Current sensor at start | Gradual increase (not instant) | [ ] |
| Log message | "Ramp complete at X% travel" | [ ] |
| Soft Start Active sensor after ramp | OFF | [ ] |

### Scenario 8.2 — Soft stop ramp approaching endstop

Observe the gate in the final approach to the open endstop.

| Check | Expected | Pass |
|-------|----------|------|
| Soft Stop Active sensor (last ~1% of travel) | ON | [ ] |
| Motor Current sensor near endstop | Reduces before endstop triggers | [ ] |
| Gate arrival at endstop | Clean stop, no audible slam | [ ] |

### Scenario 8.3 — Soft stop ramp for position target

Observe the gate approaching a 50% position target.

| Check | Expected | Pass |
|-------|----------|------|
| Soft Stop Active sensor (last ~5% before target) | ON | [ ] |
| Gate | Visibly slows before stopping | [ ] |
| Gate stops at | ~50% | [ ] |

### Scenario 8.4 — Soft start disabled during timing learning run

Precondition: Learned times reset. State = CLOSED. Send OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| Log message | "TIMING RUN ARMED" | [ ] |
| Soft Start Active sensor | Stays OFF entire run | [ ] |
| Motor current at start | Full power from the first moment | [ ] |

---

## 9 — Obstacle Detection

> **Safety:** Use a test object that will not be damaged and that the gate can safely contact. Test at 30–70% travel (well clear of the 0–10% and 90–100% near-endstop zones). Keep a hand near the Stop button throughout.

### Scenario 9.1 — First obstacle hit: gate reverses then stops

Precondition: Timing learned. Place obstacle in the gate path at approximately 50%.

Send OPEN and allow gate to contact the obstacle.

| Check | Expected | Pass |
|-------|----------|------|
| Gate stops after contact | Within 50ms (one check cycle) | [ ] |
| Log message | "OBSTACLE HIT: X.XXA > threshold at pos XX% → reversing" | [ ] |
| Gate reverses direction | Yes | [ ] |
| Reversal distance | Approximately equal to Obstacle Reverse Distance setting | [ ] |
| Log message after reversal | "Reverse complete at XX% → STOP" | [ ] |
| FSM state after reversal | STOPPED | [ ] |

### Scenario 9.2 — Second obstacle during reverse → FAULT

Position an obstacle that also blocks the reverse path (or quickly place a second object after the first hit).

Allow the gate to hit the obstacle during its reversal.

| Check | Expected | Pass |
|-------|----------|------|
| Log message | "OBSTACLE DURING REVERSE: X.XXA > threshold → FAULT" | [ ] |
| Motor | Stops immediately | [ ] |
| FSM state | FAULT | [ ] |
| Open LED / Close LED | Both solid | [ ] |

### Scenario 9.3 — Clear FAULT and resume after obstacle

Precondition: State = FAULT. Remove all obstacles.

| Check | Expected | Pass |
|-------|----------|------|
| After pressing Cycle | State = STOPPED | [ ] |
| Both LEDs | Fast flash | [ ] |
| After pressing Cycle again | State = OPENING_CYCLE, gate moves normally | [ ] |

### Scenario 9.4 — Near-endstop threshold automatically increases

Trigger an obstacle hit while the gate is in the last 10% of travel (position > 90% or < 10%).

| Check | Expected | Pass |
|-------|----------|------|
| Log shows threshold when hit | Base threshold × 1.15 | [ ] |
| Gate did not false-trigger on normal mechanical resistance near endstop | Yes | [ ] |

If the gate does false-trigger on normal resistance near the endstop, raise the base threshold via the Obstacle Threshold number entity in HA.

### Scenario 9.5 — Grace period absorbs startup inrush current

Precondition: Reduce the Obstacle Threshold entity to a low value (e.g., 0.05 A) to expose this scenario. Restore it after the test.

Send OPEN and observe the motor current spike at startup.

| Check | Expected | Pass |
|-------|----------|------|
| Obstacle triggered during first 300ms | No (grace period active) | [ ] |
| Log message at ~300ms | "Grace period ended, monitoring active" | [ ] |

### Scenario 9.6 — Obstacle suppressed during soft start

During soft start the current readings are unreliable at low PWM, so obstacle detection is suppressed.

| Check during soft start ramp | Expected | Pass |
|------------------------------|----------|------|
| Soft Start Active sensor | ON | [ ] |
| Obstacle triggered despite low threshold | No | [ ] |
| Obstacle detection activates | After Soft Start Active goes OFF | [ ] |

---

## 10 — Movement Timeout & FAULT

> To simulate a timeout, physically block or disconnect the open endstop sensor so it cannot trigger. The timeout fires at 1.5 × the learned travel time.

### Scenario 10.1 — Movement timeout triggers FAULT

Precondition: Timing learned. Open endstop disabled.

Send OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| Log message | "Movement timeout armed: Xs (travel=Ys × 1.5)" | [ ] |
| After 1.5× travel time | "MOVEMENT TIMEOUT: Xs elapsed > Xs limit → FAULT" | [ ] |
| Motor | Stops immediately | [ ] |
| FSM state | FAULT | [ ] |
| Open LED / Close LED | Both solid | [ ] |

Re-enable the endstop and clear FAULT (Cycle → STOPPED, Cycle → OPENING_CYCLE).

### Scenario 10.2 — Timeout uses fallback when timing not learned

Precondition: Learned times reset. Send OPEN.

| Check | Expected | Pass |
|-------|----------|------|
| Log message | "Timing not learned: using default open time" | [ ] |
| Timeout value in log | open_time_fallback × 1.5 | [ ] |
| Gate completes travel normally | Endstop stops it before timeout | [ ] |

### Scenario 10.3 — Two-step FAULT recovery via OPEN or CLOSE command

Precondition: State = FAULT.

| Check after first OPEN command | Expected | Pass |
|--------------------------------|----------|------|
| FSM state | STOPPED_UNKNOWN (not yet moving) | [ ] |

| Check after second OPEN command | Expected | Pass |
|---------------------------------|----------|------|
| FSM state | OPENING | [ ] |
| Gate moves normally | Yes | [ ] |

---

## 11 — Reboot & Reset Button

### Scenario 11.1 — Short press (under 3 seconds) is ignored

| Check after holding for 1–2s and releasing | Expected | Pass |
|---------------------------------------------|----------|------|
| Log message | "too short, ignoring (need 3s+)" | [ ] |
| Device | Does not reboot | [ ] |

### Scenario 11.2 — Medium hold (3–9 seconds) reboots, times preserved

Note the current learned times before testing.

| Check after holding ~4s and releasing | Expected | Pass |
|----------------------------------------|----------|------|
| Log message | "REBOOT: Button held Xs — rebooting" | [ ] |
| Device reboots | Yes | [ ] |
| Learned times after reboot | Unchanged | [ ] |

### Scenario 11.3 — Long hold (10+ seconds) resets times and reboots

| Check after holding 10+ seconds and releasing | Expected | Pass |
|------------------------------------------------|----------|------|
| Log message | "RESET: Button held Xs — resetting learned times + rebooting" | [ ] |
| Learned Open Time after reboot | 0 | [ ] |
| Learned Close Time after reboot | 0 | [ ] |
| Warning on next motion | "Timing not learned" in logs | [ ] |

### Scenario 11.4 — Reset Learned Times HA button

| Check after pressing Reset Learned Times button in HA | Expected | Pass |
|-------------------------------------------------------|----------|------|
| Learned Open Time | 0 | [ ] |
| Learned Close Time | 0 | [ ] |

---

## 12 — Home Assistant Entity Audit

Verify every entity is present, labelled correctly, and responsive. The entity prefix matches your **friendly_name** substitution (default: `gate_controller`).

| Entity | Type | Present | Responds |
|--------|------|---------|----------|
| cover.gate | Cover (open / close / stop / position slider) | [ ] | [ ] |
| sensor — State | FSM state string | [ ] | [ ] |
| sensor — Gate Position | 0–100% | [ ] | [ ] |
| sensor — Learned Open Time | Seconds | [ ] | [ ] |
| sensor — Learned Close Time | Seconds | [ ] | [ ] |
| sensor — Close Countdown | Seconds remaining | [ ] | [ ] |
| sensor — Gate Motor Current | Amperes | [ ] | [ ] |
| binary_sensor — Open Endstop | Reflects physical switch | [ ] | [ ] |
| binary_sensor — Close Endstop | Reflects physical switch | [ ] | [ ] |
| binary_sensor — Auto Close Armed | ON during OPEN_CYCLE countdown | [ ] | [ ] |
| button — Reset Learned Times | Clears both learned times | [ ] | [ ] |
| button — Gate Cycle | Software Cycle trigger | [ ] | [ ] |
| number — Obstacle Threshold | Adjustable threshold (Amperes) | [ ] | [ ] |
| number — Obstacle Reverse Distance | Reverse distance (0.0–1.0) | [ ] | [ ] |
| sensor — WiFi Signal Strength | dBm (diagnostic) | [ ] | [ ] |
| sensor — Uptime | Human-readable (diagnostic) | [ ] | [ ] |
| binary_sensor — Status | Online / offline (diagnostic) | [ ] | [ ] |
| switch — Restart | Remote restart (diagnostic) | [ ] | [ ] |

---

## 13 — LED Pattern Verification

Use the scenarios from prior sections to enter each state and verify the physical panel LEDs.

| State | Open LED (GPB0) | Close LED (GPB1) | Pass |
|-------|-----------------|------------------|------|
| OPEN | Solid | Off | [ ] |
| CLOSED | Off | Solid | [ ] |
| OPENING | Fast flash (2 Hz) | Off | [ ] |
| OPENING_CYCLE | Fast flash (2 Hz) | Off | [ ] |
| CLOSING | Off | Fast flash (2 Hz) | [ ] |
| OPEN_CYCLE | Slow pulse (1 Hz) | Off | [ ] |
| STOPPED | Fast flash (2 Hz) | Fast flash (2 Hz) | [ ] |
| PAUSED_FROM_OPENING | Fast flash (2 Hz) | Fast flash (2 Hz) | [ ] |
| PAUSED_FROM_CLOSING | Fast flash (2 Hz) | Fast flash (2 Hz) | [ ] |
| FAULT | Solid | Solid | [ ] |
| STOPPED_UNKNOWN | Fast flash | Fast flash (opposite phase) | [ ] |
| Jog opening (any FSM state) | Fast flash (2 Hz) | Off | [ ] |
| Jog closing (any FSM state) | Off | Fast flash (2 Hz) | [ ] |

---

## 14 — Safety Architecture

### Scenario 14.1 — Endstop kills motor before FSM processes the transition

Allow the gate to open fully and trigger the open endstop normally.

| Check | Expected | Pass |
|-------|----------|------|
| Log order | "emergency motor kill" appears before FSM transition message | [ ] |
| Motor Current sensor at endstop trigger | Drops to 0 immediately | [ ] |
| Visible gate overrun | None | [ ] |

### Scenario 14.2 — 100ms watchdog interval is independent of scripts

Manually trigger the open endstop signal (hold the switch closed) while the gate is moving open.

| Check | Expected | Pass |
|-------|----------|------|
| Motor stops | Within 100ms of endstop activation | [ ] |
| Stop occurs independently | Even if scripts are mid-execution | [ ] |

### Scenario 14.3 — motor_kill flag holds zero PWM after a stop

Stop the gate mid-motion and watch the Motor Current sensor closely.

| Check | Expected | Pass |
|-------|----------|------|
| Motor Current after stop | Drops to 0 and stays at 0 | [ ] |
| Any brief current spike after stop | None (motor_kill interlock effective) | [ ] |

### Scenario 14.4 — Endstop terminates soft start / soft stop loops

Allow a stop event (endstop or emergency stop) to fire while a ramp script is mid-loop.

| Check | Expected | Pass |
|-------|----------|------|
| Ramp scripts self-terminate | Yes (level-triggered check at top of each loop) | [ ] |
| No continued PWM after stop | Confirmed by current sensor | [ ] |

---

## 15 — Connectivity & Status LED

The status LED is the onboard blue LED on the Wemos D1 Mini32 (GPIO2).

| State | Status LED | Pass |
|-------|-----------|------|
| WiFi disconnected | Fast blink (500ms on / 500ms off) | [ ] |
| WiFi connected, HA API disconnected | Solid ON | [ ] |
| WiFi connected, HA API connected | Slow breathing glow (~2s cycle) | [ ] |

### Scenario 15.4 — Fallback AP on WiFi failure

Boot the device with no configured WiFi available.

| Check | Expected | Pass |
|-------|----------|------|
| SSID visible | `gate-controller` | [ ] |
| Captive portal at 192.168.4.1 | Yes | [ ] |

### Scenario 15.5 — OTA firmware update

Trigger an OTA update from the ESPHome dashboard.

| Check | Expected | Pass |
|-------|----------|------|
| Update completes | Yes | [ ] |
| Device reconnects after reboot | Yes | [ ] |
| FSM initial state correct after OTA | Matches gate physical position | [ ] |

---

## 16 — Edge Cases

### Scenario 16.1 — CYCLE from STOPPED_UNKNOWN sends gate to close endstop

Precondition: State = STOPPED_UNKNOWN (boot with gate mid-travel).

| Check | Expected | Pass |
|-------|----------|------|
| FSM state after Cycle | CLOSING | [ ] |
| Gate travels to close endstop | State = CLOSED | [ ] |
| Position now known | Normal operation resumes | [ ] |

### Scenario 16.2 — Rapid Cycle presses remain predictable

Press Cycle 3 times in rapid succession (~200ms apart) from State = CLOSED.

| Check | Expected | Pass |
|-------|----------|------|
| Each press advances through valid FSM transitions | Yes | [ ] |
| No double-motion, no stuck state, no crash | Yes | [ ] |

### Scenario 16.3 — Software Cycle button in HA is identical to panel button

| Check from State = CLOSED | Expected | Pass |
|---------------------------|----------|------|
| Press Gate Cycle button in HA | State = OPENING_CYCLE | [ ] |

### Scenario 16.4 — Position tracking corrects to exact values at endstops

Run gate mid-travel repeatedly to allow position drift, then close and open fully.

| Check | Expected | Pass |
|-------|----------|------|
| Gate Position sensor at close endstop | Resets to exactly 0% | [ ] |
| Gate Position sensor at open endstop | Resets to exactly 100% | [ ] |

---

## Summary

| Section | Description | Pass |
|---------|-------------|------|
| 1 | Initial state detection on boot | [ ] |
| 2 | Timing learning | [ ] |
| 3 | Basic open / close / direction reversal | [ ] |
| 4 | Cycle button & auto-close | [ ] |
| 5 | Hold-to-operate (jog) | [ ] |
| 6 | Stop & pause mid-travel | [ ] |
| 7 | Position control | [ ] |
| 8 | Soft start & soft stop | [ ] |
| 9 | Obstacle detection | [ ] |
| 10 | Movement timeout & FAULT | [ ] |
| 11 | Reboot & reset button | [ ] |
| 12 | Home Assistant entity audit | [ ] |
| 13 | LED pattern verification | [ ] |
| 14 | Safety architecture | [ ] |
| 15 | Connectivity & status LED | [ ] |
| 16 | Edge cases | [ ] |

Tester: _____________________________ &nbsp;&nbsp; Date: _______________ &nbsp;&nbsp; Firmware: _______________
