# ESPHome Gate Controller

A feature-rich smart gate/door controller for Home Assistant, built with a Finite State Machine (FSM) architecture. Runs on an ESP32 (Wemos D1 Mini32) with an MCP23017 I/O expander for control panel I/O and an INA260 current sensor for obstacle detection.

## Features

- **Finite State Machine** — 11 states, clean predictable transitions, full HA visibility
- **Soft Start / Soft Stop** — smooth motor ramp-up and ramp-down to reduce mechanical stress
- **Auto-Close** — configurable countdown timer that closes the gate automatically when opened via the cycle button
- **One-Button (Cycle) Control** — full gate operation from a single momentary button
- **Hold-to-Operate (Jog)** — direct motor control buttons (dead man's switch) that bypass the FSM. Great for gate initial positioning, and testing the endstop function. 
- **Position Tracking** — time-based position estimation with automatic correction at endstops
- **Position Control** — move the gate to any position via the Home Assistant cover slider
- **Timing Learning** — automatically measures full travel time on the first run; persists across reboots
- **Obstacle Detection** — current-based detection with configurable threshold and automatic reversal
- **Movement Timeout** — watchdog that faults the controller if travel exceeds 1.5× expected time
- **Control Panel** — 5 buttons and 2 LEDs for standalone operation without a phone or PC

## Safety Architecture

Multiple independent layers prevent the gate from overrunning its endstops:

1. **`motor_kill` global interlock** — `apply_pwm` always writes zero when this flag is set, regardless of `pwm_level`
2. **Explicit `script.stop` in endstop handlers** — all motion scripts are terminated before FSM transition
3. **`motor_kill` check in every while-loop condition** — loops self-terminate if the flag is set
4. **Level-triggered endstop checks inside ramp loops** — safety check at the top of each iteration catches edge cases
5. **100ms interval watchdog** — completely independent hardware check; worst-case overrun is 100ms

## Hardware

| Component | Detail |
|-----------|--------|
| Microcontroller | Wemos D1 Mini32 (ESP32) |
| I/O Expander | MCP23017 at I2C address 0x20 |
| Current Sensor | INA260 at I2C address 0x40 |
| Motor Driver | H-bridge with PWM inputs |
| Motor PWM Pins | GPIO16 (open direction), GPIO17 (close direction) |
| I2C Bus | SDA: GPIO21, SCL: GPIO22 |
| Status LED | GPIO2 (onboard blue LED) |
| Power LED | Wired directly to 3.3V (always on) |

### Control Panel Wiring (MCP23017)

**Port A — Inputs (GPA0–GPA7, all pulled up, active low):**

| Pin | Function |
|-----|----------|
| GPA0 | Open endstop switch |
| GPA1 | Close endstop switch |
| GPA2 | Cycle button (momentary) |
| GPA3 | Open button (hold-to-operate) |
| GPA4 | Close button (hold-to-operate) |
| GPA5 | Stop button (emergency stop, momentary) |
| GPA6 | Reboot/Reset button (3 s = reboot, 10 s = reset learned times + reboot) |
| GPA7 | Spare |

**Port B — LED Outputs (GPB0–GPB1):**

| Pin | LED | Behavior |
|-----|-----|----------|
| GPB0 | Open LED | See table below |
| GPB1 | Close LED | See table below |
| GPB2–GPB7 | Spare | — |

**LED behavior:**

| Gate state | Open LED (GPB0) | Close LED (GPB1) |
|------------|-----------------|------------------|
| OPEN | Solid | Off |
| CLOSED | Off | Solid |
| Opening / Opening with auto-close | Fast flash (2 Hz) | Off |
| Closing | Off | Fast flash (2 Hz) |
| Auto-close countdown (at open endstop) | Slow pulse (1 Hz) | Off |
| Stopped mid-travel | Fast flash (2 Hz) | Fast flash (2 Hz) |
| FAULT | Solid | Solid |
| Boot / position unknown | Alternating fast flash | Alternating fast flash |

## Repository Structure

```
gate-controller/
├── devices/
│   └── gate-controller.yaml   # Main device configuration
├── packages/
│   ├── esp_common.yaml        # Shared ESPHome base (WiFi, API, OTA, diagnostics)
│   └── wemos_d1_mini32.yaml   # Hardware package (board, status LED, connectivity callbacks)
└── secrets.example.yaml       # Template for required secrets
```

## Setup

### 1. Secrets

Copy `secrets.example.yaml` to `secrets.yaml` and fill in your values:

```yaml
wifi_ssid: "YourNetwork"
wifi_password: "YourPassword"
ota_password: "a-strong-random-password"
api_encryption_key: "a-base64-key-from-esphome-keygen"
wifi_domain: ".local"
```

### 2. Clone into your ESPHome config directory

The default ESPHome config directory on Home Assistant is `/config/esphome/`. The device YAML references the packages using absolute paths, so place the repo here:

```
/config/esphome/repos/gate-controller/
```

If you clone it elsewhere, update the `packages:` block in `devices/gate-controller.yaml`:

```yaml
packages:
  - !include /path/to/packages/esp_common.yaml
  - !include /path/to/packages/wemos_d1_mini32.yaml
```

### 3. Review substitutions

Check the `substitutions:` block at the top of `devices/gate-controller.yaml` and adjust any values for your installation:

```yaml
substitutions:
  device_name: gate-controller      # ESPHome device name (lowercase, dashes ok)
  friendly_name: Gate Controller    # Display name in Home Assistant
  open_pin: GPIO16                  # PWM output for open direction
  close_pin: GPIO17                 # PWM output for close direction
  i2c_sda: GPIO21
  i2c_scl: GPIO22
```

### 4. Create the required Home Assistant entity

The auto-close feature reads its countdown duration from an `input_number` in Home Assistant:

```
input_number.entry_gate_duration
```

Create it before flashing (min: 1, max: 300, step: 1, unit: seconds).

### 5. Flash and first run

1. Flash the firmware to your ESP32
2. On boot the controller checks the endstops and sets its initial FSM state
3. Run the gate end-to-end once (closed → open → closed) so it can learn travel times
4. After the first full learning run, timing is stored in flash and position tracking is accurate

## Home Assistant Entities

| Entity | Description |
|--------|-------------|
| `cover.gate` | Main gate control (open / close / stop / position) |
| `sensor…_state` | Current FSM state |
| `sensor…_gate_position` | Estimated position (%) |
| `sensor…_learned_open_time` | Measured full-open travel time (s) |
| `sensor…_learned_close_time` | Measured full-close travel time (s) |
| `sensor…_close_countdown` | Auto-close seconds remaining |
| `sensor.gate_motor_current` | Motor current (A) |
| `binary_sensor.gate_open_endstop` | Open endstop state |
| `binary_sensor.gate_close_endstop` | Close endstop state |
| `binary_sensor…_auto_close_armed` | Auto-close active flag |
| `button…_reset_learned_times` | Clears stored travel times |
| `button…_gate_cycle` | Software cycle button |
| `number…_obstacle_threshold` | Obstacle detection current threshold (A) |
| `number…_obstacle_reverse_distance` | How far to reverse on obstacle (%) |

> Entity IDs use the `friendly_name` substitution as a prefix (default: `gate_controller`).

## FSM States

| State | Meaning |
|-------|---------|
| `STOPPED_UNKNOWN` | Boot state — position not yet determined from endstops |
| `OPENING` | Moving toward open endstop (no auto-close) |
| `OPENING_CYCLE` | Moving toward open endstop (auto-close will arm on arrival) |
| `OPEN` | At open endstop, no auto-close |
| `OPEN_CYCLE` | At open endstop, auto-close countdown active |
| `CLOSING` | Moving toward close endstop |
| `CLOSED` | At close endstop |
| `PAUSED_FROM_OPENING` | Stopped mid-travel while opening |
| `PAUSED_FROM_CLOSING` | Stopped mid-travel while closing |
| `STOPPED` | Stopped by an explicit STOP command |
| `FAULT` | Fault condition (obstacle on reverse, movement timeout) |

## Troubleshooting

**Gate doesn't move**
- Check the FSM state sensor — if it shows `FAULT`, send CYCLE to clear it, then command motion again
- Verify the endstop binary sensors show the correct state
- Check ESPHome logs for `motor_start` entries and the PWM level being applied

**Position tracking is inaccurate**
- Use the Reset Learned Times button and run a full endstop-to-endstop travel
- Watch for `FALLBACK` warnings in the logs — these appear before timing has been learned
- Set `open_time_fallback` / `close_time_fallback` close to your gate's actual travel time to improve accuracy before the first learning run

**Soft start/stop not working**
- During timing learning runs, soft start is intentionally disabled for an accurate speed-constant measurement
- Check the `soft_start_active` / `soft_stop_active` diagnostic binary sensors in HA

**Auto-close not triggering**
- Verify `input_number.entry_gate_duration` exists in HA
- Auto-close only counts down while the gate is physically at the open endstop and the FSM is in `OPEN_CYCLE`
- Any manual intervention (panel button, HA command) cancels the countdown

**Obstacle detection false positives**
- Raise the threshold via the Obstacle Threshold number entity
- The 300 ms grace period after motor start absorbs inrush current spikes — if current is still spiking, verify motor wiring
- Near endstops (0–10% and 90–100% travel), the threshold is automatically raised 15% to account for normal mechanical resistance

**Movement timeout / FAULT**
- Check for a mechanical obstruction, failed endstop, or belt/chain slip
- If the endstop is working normally, the timeout may be shorter than your gate needs — reset learned times and do a clean learning run
- Clear fault with CYCLE (then CYCLE again to move), or use the OPEN/CLOSE buttons

## Changelog

### v1.0 — Initial Release

Full-featured FSM-based gate controller with soft start/stop, time-based position tracking, timing learning, one-button cycle control with auto-close, hold-to-operate jog buttons, current-based obstacle detection with automatic reversal, movement timeout watchdog, and a five-layer endstop overrun safety architecture.
