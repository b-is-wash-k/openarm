# OpenArm Session — Step 6: Bug Fix (Gripper Overshoot) + Sim Viewer in All Scripts
### What went wrong, why, and how all three scripts were fixed
**AIR Lab, NC State University**

---

## The Incident: Motor 8 Vibrating

### What happened
```
> 8 0.1
  → Motor 8: moving to 0.100 rad
  [arm starts vibrating — robot powered off immediately]
```

### Why it happened

Motor 8 is the **gripper**. Its physical limits are:
```
min_angle_left = -45° = -0.785 rad
max_angle_left =   0° =  0.000 rad   ← HARD STOP at 0
```

The initial position was `-0.274 rad` (gripper slightly open).
The old clamping only checked `initial ± 1.5 rad`, so `0.1 rad` passed the check.

But `0.1 rad` is **above the physical stop** of the gripper (max = 0 rad).
The motor tried to push the gripper finger past where it can mechanically go.
The motor could not reach the target, kept trying harder, motor fought the wall → vibration.

```
Initial:    -0.274 rad
Commanded:   0.100 rad  ← above the 0.0 rad physical ceiling
Limit:       0.000 rad  ← motor hits this and starts fighting

Old clamp:   initial ± 1.5  →  [-1.774, 1.274]  WRONG — doesn't know about physical limits
New clamp:   MOTOR_CONFIGS limits  →  [-0.785, 0.000]  CORRECT
```

### The fix

All scripts now compute limits directly from `MOTOR_CONFIGS` (the same source of truth
that the gravity compensation and robot config use):

```python
def joint_limits_rad(motor_idx: int, side: str) -> tuple[float, float]:
    cfg = MOTOR_CONFIGS[motor_idx]
    if side == "left":
        lo = math.radians(cfg.min_angle_left)   # -45° → -0.785 rad
        hi = math.radians(cfg.max_angle_left)   #   0° →  0.000 rad
    ...
    return (min(lo, hi), max(lo, hi))
```

With this fix, commanding motor 8 to `0.1 rad` now prints:
```
⚠ Clamped 0.100 → 0.000 rad  (joint limit: [-0.785, 0.000])
```

---

## Full Joint Limits Reference (left arm)

| Motor | Joint | Min (°) | Max (°) | Min (rad) | Max (rad) |
|---|---|---|---|---|---|
| 1 | shoulder rotation | -200 | +80 | -3.491 | +1.396 |
| 2 | shoulder flex | -190 | +10 | -3.316 | +0.175 |
| 3 | upper arm rotation | -90 | +90 | -1.571 | +1.571 |
| 4 | elbow flex | 0 | +140 | 0.000 | +2.443 |
| 5 | forearm rotation | -90 | +90 | -1.571 | +1.571 |
| 6 | wrist flex | -45 | +45 | -0.785 | +0.785 |
| 7 | wrist rotation | -90 | +90 | -1.571 | +1.571 |
| **8** | **gripper** | **-45** | **0** | **-0.785** | **0.000** |

⚠ **Gripper note:** The gripper can only CLOSE (negative direction from 0).
Any positive target for motor 8 will hit the mechanical stop immediately.

---

## What Changed in the Three Scripts

All three scripts now have two major improvements:

### 1. Correct joint limit clamping

```
Old:  clamp to  initial ± 1.5 rad   (doesn't know about mechanical limits)
New:  clamp to  MOTOR_CONFIGS limits (degrees → radians, per arm side)
```

For oscillation scripts, the amplitude is also automatically clamped:
```python
max_down    = initial[i] - lo          # how far we can go below initial
max_up      = hi - initial[i]          # how far we can go above initial
safe_amp    = min(desired_amp, max_down, max_up)
```

### 2. MuJoCo viewer added to all scripts

All three scripts now show the virtual robot twin in the MuJoCo viewer
while the real hardware is running.

The architecture is the same in all scripts:

```
Background thread (CAN):
    asyncio loop → sends MIT commands → reads back actual positions
    → writes to SharedState

Main thread (viewer):
    mujoco.viewer.launch_passive(sim.model, sim.data)
    while viewer.is_running():
        actual = shared.get_actual()
        sim.set_left_arm_positions(actual[:7])
        sim.set_left_gripper_position(actual[7])
        viewer.sync()
```

---

## How to Run the Fixed Scripts

### Script 1 — Interactive control with viewer
```bash
python examples/hardware_motor_control.py --port robot_l:left
```
- MuJoCo window opens showing the arm
- Type in terminal: `3 -0.5` → motor 3 moves, viewer updates live
- Out-of-limit values are clamped with a warning message
- Type `q` or close the viewer to stop

### Script 2 — Single motor oscillation with viewer
```bash
python examples/hardware_single_motor_oscillation.py --port robot_l:left --motor 3
```
- Motor 3 oscillates, all others hold
- Amplitude auto-clamped to fit within joint limits
- Viewer shows real positions updating live

### Script 3 — All joints wave oscillation with viewer
```bash
python examples/hardware_all_joints_oscillation.py --port robot_l:left
```
- All joints ripple in a wave
- Each joint's amplitude individually clamped to its own limits
- Viewer shows the whole arm moving in real time

### Stopping all scripts
- Close the MuJoCo viewer window, **or**
- Press `Ctrl+C` in the terminal

---

## Safety Rules After This Incident

1. **Never command a positive value to motor 8 (gripper)** — its max is 0 rad (fully closed).
   The gripper only opens by going negative (toward -0.785 rad).

2. **Always check the joint limits table before commanding new positions.**
   Motor 4 (elbow) also has a one-sided limit: it can only flex from 0° to 140°.
   Negative elbow values will fight the mechanical stop.

3. **Trust the scripts' automatic clamping** — but understand WHY a value gets clamped.

4. **Start with small movements** (±0.1 rad) when trying a new joint for the first time.

---

## Complete File List

```
examples/
├── simulation_demo.py                   Step 2 — MuJoCo sim only
├── hardware_oscillation_demo.py         Step 3/4 — joint 1 only, 1 or 2 arms
├── hardware_sim_mirror.py               Step 4 — hardware + sim viewer
├── hardware_motor_control.py            Step 5/6 — interactive + viewer + limit fix
├── hardware_single_motor_oscillation.py Step 5/6 — one motor + viewer + limit fix
└── hardware_all_joints_oscillation.py   Step 5/6 — all joints wave + viewer + limit fix
```

---

*Written after Step 6 session — gripper incident, limit fix, and viewer integration. AIR Lab, NC State University.*
