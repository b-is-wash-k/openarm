# OpenArm Session — Step 5: Interactive Control + Per-Motor Oscillation
### Three new tools: control any joint, oscillate one, oscillate all
**AIR Lab, NC State University**

---

## Three Scripts Built This Session

| Script | What it does |
|---|---|
| `hardware_motor_control.py` | Type a motor number + angle → that joint moves |
| `hardware_single_motor_oscillation.py` | Pick one motor to oscillate, rest hold still |
| `hardware_all_joints_oscillation.py` | All joints oscillate as a ripple wave |

---

## Script 1 — Interactive Motor Control

### How to run
```bash
python examples/hardware_motor_control.py --port robot_l:left
```

### What you see
```
Enabling 8 motors on robot_l (left arm)...
Initial positions (rad): [0.013, -0.008, -0.298, -0.015, -1.159, -0.249, -0.005, -0.274]

──────────────────────────────────────────────────
Interactive motor control ready.
  Type:  <motor_id> <position_rad>   then Enter
  Type:  q                            to quit
  Motors available: 1 – 8
  Max displacement from start: ±1.5 rad
──────────────────────────────────────────────────

> 1 0.5          ← move motor 1 (shoulder rotation) to 0.5 rad
  → Motor 1: moving to 0.500 rad

> 3 -0.3         ← move motor 3 (upper arm rotation) to -0.3 rad
  → Motor 3: moving to -0.300 rad

> 1 0            ← move motor 1 back to 0
  → Motor 1: moving to 0.000 rad

> q              ← quit
Disabling all motors...
Done.
```

### How it works internally

Two threads run simultaneously:

```
Main thread (asyncio):
    hold_loop()
        Every 0.02s → sends MIT control to ALL motors with current targets
        Keeps all motors engaged and holding their positions

Input thread (blocking):
    input_loop()
        Waits for user to press Enter
        Parses "motor_id position_rad"
        Updates SharedTargets[motor_index] = new_position

SharedTargets (thread-safe with lock):
    hold_loop reads targets  ──┐
                               ├── same data, protected by threading.Lock
    input_loop writes targets ─┘
```

The `threading.Lock` is like a traffic light: only one thread can touch
the targets array at a time. This prevents garbled data.

### Safety: MAX_DELTA clamp
```
If you type:  1 5.0   (5 radians = 286° — dangerously far)
Script clamps it to:  initial[0] + 1.5 rad  (max allowed)
  ⚠ Clamped to 1.513 rad  (limit: [-1.487, 1.513])
```

---

## Script 2 — Single Motor Oscillation

### How to run
```bash
# Oscillate motor 3 (upper arm rotation) with defaults
python examples/hardware_single_motor_oscillation.py --port robot_l:left --motor 3

# Custom amplitude and speed
python examples/hardware_single_motor_oscillation.py --port robot_l:left --motor 4 --amplitude 0.2 --frequency 0.003
```

### What you see
```
Enabling 8 motors on robot_l (left arm)...
Initial positions (rad): [0.013, -0.008, -0.298, ...]

Oscillating motor 3  (J3)
  Amplitude : 0.3 rad  (~17.2°)
  All other motors hold position.

Press  Q  to stop.
```

### Arguments

| Argument | Default | Meaning |
|---|---|---|
| `--port` | required | e.g. `robot_l:left` |
| `--motor` | required | which motor to oscillate (1–8) |
| `--amplitude` | `0.3` | how far it swings in radians |
| `--frequency` | `0.005` | speed (higher = faster) |

### Difference from `hardware_oscillation_demo.py`
The original demo was hardcoded to joint 1 only.
This script lets you pick **any** motor by number.

```bash
# Try each joint one at a time to learn what each one does:
python ... --motor 1   # shoulder rotates left/right
python ... --motor 2   # shoulder raises up/down
python ... --motor 3   # upper arm twists
python ... --motor 4   # elbow bends
python ... --motor 5   # forearm twists
python ... --motor 6   # wrist bends
python ... --motor 7   # wrist rotates
python ... --motor 8   # gripper opens/closes
```

---

## Script 3 — All Joints Wave Oscillation

### How to run
```bash
# One arm
python examples/hardware_all_joints_oscillation.py --port robot_l:left

# Both arms
python examples/hardware_all_joints_oscillation.py --port robot_l:left --port robot_r:right

# Faster wave
python examples/hardware_all_joints_oscillation.py --port robot_l:left --frequency 0.008
```

### What you see on startup
```
Starting wave oscillation — ALL joints moving.
  Frequency : 0.005 cycles/step
  Phase     : 45° offset between each joint (ripple wave)

  Joint  Amplitude    Phase
    J1     ±0.30 rad     0°
    J2     ±0.20 rad    45°
    J3     ±0.20 rad    90°
    J4     ±0.15 rad   135°
    J5     ±0.20 rad   180°
    J6     ±0.15 rad   225°
    J7     ±0.15 rad   270°
    J8     ±0.05 rad   315°

Press  Q  to stop.
```

### What the wave motion looks like

Because each joint has a 45° phase offset from the previous one:

```
At t=0:    J1 at peak  →  J2 slightly behind  →  J3 more behind  → ...
At t=1/8:  J1 descending, J2 at peak, J3 slightly behind, ...
At t=2/8:  J1 at zero,  J2 descending, J3 at peak, ...
```

This creates a **ripple wave** through the arm from shoulder to gripper.
All 8 joints are in continuous sinusoidal motion, each slightly delayed.

### Why different amplitudes per joint?

| Joint | Amplitude | Reason |
|---|---|---|
| J1 (shoulder) | 0.30 rad (17°) | Large motor, lots of room to move |
| J2 (shoulder) | 0.20 rad (11°) | Medium — this lifts the whole arm |
| J3–J5 | 0.20 rad (11°) | Mid-arm joints have good range |
| J4 (elbow) | 0.15 rad (9°) | Smaller — elbow takes the weight of forearm |
| J6–J7 (wrist) | 0.15 rad (9°) | Smaller range joint |
| J8 (gripper) | 0.05 rad (3°) | Very small — just a slight open/close |

---

## Complete File List After This Session

```
examples/
├── simulation_demo.py                  ← Step 2: MuJoCo simulation only
├── hardware_oscillation_demo.py        ← Step 3/4: oscillate joint 1 (1 or 2 arms)
├── hardware_sim_mirror.py              ← Step 4: hardware + MuJoCo twin
├── hardware_motor_control.py           ← Step 5 NEW: interactive joint control
├── hardware_single_motor_oscillation.py← Step 5 NEW: pick any motor to oscillate
└── hardware_all_joints_oscillation.py  ← Step 5 NEW: all joints wave motion
```

---

## Quick Command Reference

```bash
# Move any motor interactively
python examples/hardware_motor_control.py --port robot_l:left

# Oscillate one specific motor
python examples/hardware_single_motor_oscillation.py --port robot_l:left --motor 3
python examples/hardware_single_motor_oscillation.py --port robot_l:left --motor 4 --amplitude 0.2

# Wave all joints at once
python examples/hardware_all_joints_oscillation.py --port robot_l:left
python examples/hardware_all_joints_oscillation.py --port robot_l:left --port robot_r:right --frequency 0.008
```

---

## What Comes Next

```
1. Gravity compensation + live MuJoCo mirror
   → move the arm by hand, watch it in simulation

2. Record joint positions during hand-guided motion → save to CSV/numpy

3. Replay recorded trajectory on hardware

4. Use inverse kinematics (IKPy module) to compute joint angles
   from a desired (x, y, z) end-effector position
```

---

*Written after Step 5 session at AIR Lab, NC State University.*
