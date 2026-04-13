# OpenArm Session — Step 3: Running the Simulation Motion on Real Hardware
### Translating the MuJoCo demo to physical motors
**AIR Lab, NC State University**

---

## Goal of This Step

In Step 2, the simulation showed both arms oscillating at joint 1 (shoulder rotation).
The goal here is to run the **same sinusoidal motion on the real physical arm** using the
CAN bus and Damiao motors.

---

## The Key Question: Can the Sim Code Talk to the Real Robot Directly?

**No — they use different APIs:**

```
Simulation (MuJoCo):
  sim.set_left_arm_position_control([joint1_target, ...])
  → writes to MuJoCo data arrays internally

Real robot (CAN bus / Damiao):
  await arm.control_mit(kp=..., kd=..., q=[targets], dq=0, tau=0)
  → sends CAN message over robot_l interface to physical motors
```

But the **concept is the same**: send a target joint angle to joint 1 every loop step,
with PD gains controlling how hard the motor tracks that target.

---

## What MIT Mode Means

MIT mode = **M**IT **I**mpedance Con**t**rol mode.
It was originally developed at MIT for compliant legged robots.

You send four values per motor every loop cycle:

| Parameter | Meaning | Value used in demo |
|---|---|---|
| `kp` | How hard the motor tries to reach the target angle | 20.0 |
| `kd` | Damping — resists fast movements to prevent oscillation | 3.0 |
| `q`  | Target joint angle (radians) | sinusoidal for J1, held for J2–J8 |
| `dq` | Target velocity | 0 (we want the arm to hold position) |
| `tau`| Direct torque offset | 0 (no extra push, just PD control) |

The motor computes:  `torque = kp*(q_target - q_actual) + kd*(dq_target - dq_actual) + tau`

This is exactly the same as the simulation's `set_left_arm_position_control()` which uses
`kp=100, kd=10` internally.

---

## Safety Choices Made for the Real Robot

The simulation used `amplitude = 0.5 rad`. On the real arm we use **smaller, safer values**:

| Parameter | Simulation | Real robot |
|---|---|---|
| Amplitude | 0.5 rad (28°) | **0.3 rad (17°)** |
| Oscillation starts from | zero position | **current position** (wherever the arm is) |
| Other joints | hold last position | **hold their exact start positions** |

**Why start from current position?**
In simulation, the arm always starts at zero.
On the real robot, the arm might be resting at any angle.
If you command "go to 0.3 rad" from a position of 2.0 rad, the arm will snap violently.
Instead, the script reads where each joint is when it starts and oscillates *around that point*.

---

## What Was Built: `hardware_oscillation_demo.py`

A new script was written at:
```
examples/hardware_oscillation_demo.py
```

**What it does step by step:**
```
1. Parses --port robot_l:left  (CAN interface + which arm side)
2. Finds the robot_l socket CAN interface
3. Detects which of the 8 motors are responding
4. Creates Motor objects (matching MOTOR_CONFIGS in openarm/damiao/config.py)
5. Enables all motors
6. Sets control mode to MIT
7. Reads starting position of every joint
8. Loop:
     joint1_target = initial_pos[0] + 0.3 * sin(step * 0.005 * 2π)
     all others    = hold at initial_pos[i]
     → sends arm.control_mit(kp=20, kd=3, q=targets, dq=0, tau=0)
     → sleeps 0.01s (100 Hz loop)
9. Press Q → disables all motors → exits
```

---

## Motor Hardware Reference (from `openarm/damiao/config.py`)

| Joint | Name | Motor type | slave_id | master_id | Range (left arm) |
|---|---|---|---|---|---|
| 1 | J1 — shoulder rotation | DM8009 | 0x01 (1) | 0x11 (17) | -200° to +80° |
| 2 | J2 — shoulder flex | DM8009 | 0x02 (2) | 0x12 (18) | -190° to +10° |
| 3 | J3 — upper arm rotation | DM4340 | 0x03 (3) | 0x13 (19) | -90° to +90° |
| 4 | J4 — elbow flex | DM4340 | 0x04 (4) | 0x14 (20) | 0° to +140° |
| 5 | J5 — forearm rotation | DM4310 | 0x05 (5) | 0x15 (21) | -90° to +90° |
| 6 | J6 — wrist flex | DM4310 | 0x06 (6) | 0x16 (22) | -45° to +45° |
| 7 | J7 — wrist rotation | DM4310 | 0x07 (7) | 0x17 (23) | -90° to +90° |
| 8 | J8 — gripper | DM4310 | 0x08 (8) | 0x18 (24) | -45° to 0° |

Joint 1 (J1) uses the **DM8009** — the largest, most powerful motor.
Its ±200° range gives plenty of room for a small ±17° oscillation.

---

## How to Run

### Step 1 — Bring up the CAN interface
```bash
sudo ip link set robot_l down
sudo ip link set robot_l up type can bitrate 1000000
```

### Step 2 — Run the oscillation demo
```bash
cd ~/OPEN_ARM/openarm
python examples/hardware_oscillation_demo.py --port robot_l:left
```

### Expected output
```
Enabling 8 motors on robot_l (left arm)...
Initial joint positions (rad): [0.012, -0.034, 0.001, 0.521, -0.003, 0.011, 0.002, -0.015]

Starting oscillation on joint 1 (shoulder rotation).
  Amplitude : 0.3 rad  (~17.2 degrees)
  All other joints hold their starting positions.

Press  Q  to stop.
```

Then joint 1 slowly swings back and forth ±17°.
The rest of the arm stays frozen.

### Step 3 — Stop
Press **Q** in the terminal.
The script disables all motors and exits.

⚠️ Always hold the arm before pressing Q — motors will go limp immediately.

---

## Comparison: Simulation vs Hardware

| Aspect | Simulation | Real hardware |
|---|---|---|
| Control method | `sim.set_left_arm_position_control()` | `arm.control_mit(kp, kd, q, dq, tau)` |
| Physics | MuJoCo integrates physics | Real motor physics |
| Starting position | Always zero | Wherever the arm actually is |
| Safety limits | None (sim won't hurt anything) | Must use small amplitude + start from current pos |
| Feedback | `sim.get_left_arm_positions()` | `state.position` from motor reply on CAN bus |
| Loop rate | As fast as MuJoCo can run | 100 Hz (0.01s sleep) |

---

## What Comes Next

```
1. Extend to both arms simultaneously (--port robot_l:left --port robot_r:right)
2. Oscillate multiple joints at different frequencies
3. Combine with gravity compensation (so the arm feels light while oscillating)
4. Record the joint angles during the motion → build a trajectory
5. Replay that trajectory: arm repeats the motion automatically
```

---

*Written after Step 3 session — translating simulation to real hardware at AIR Lab, NC State University.*
