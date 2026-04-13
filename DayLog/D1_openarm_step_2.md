# OpenArm Session — Step 2: Running the MuJoCo Simulation
### What happened, what went wrong, and what we learned
**AIR Lab, NC State University**

---

## Goal of This Step

After completing Step 1 (CAN bus setup + motor enable/gravity compensation on real hardware),
the next step was to run the **MuJoCo physics simulation** — a virtual version of the robot
you can experiment with without needing the physical arm powered on.

---

## What Is MuJoCo?

MuJoCo stands for **Mu**lti-**Jo**int dynamics with **Co**ntact.
It is a fast physics simulator used widely in robotics research.

```
Real robot:         Python command → CAN bus → Damiao motors → physical arm moves
Simulation:         Python command → MuJoCo → virtual arm moves on screen
```

The simulation uses the same Python API as the real arm.
This means you can develop and test code in simulation before running it on hardware.

---

## The Environment Problem

The terminal was opened without activating the conda environment.
When `pip install mujoco` was run, it hit the system Python instead of the openarm env:

```
error: externally-managed-environment
× This environment is externally managed
```

**Why this happened:**
The system Python (at `/usr/bin/python`) is locked by the OS.
The conda `openarm` env has its own separate Python at:
```
/home/air-lab-ncsu/anaconda3/envs/openarm/bin/python
```

**Fix:** Always activate the conda environment first:
```bash
conda activate openarm
which python
# → /home/air-lab-ncsu/anaconda3/envs/openarm/bin/python  ✅
```

---

## The Discovery: MuJoCo Was Already Installed

After activating the correct environment and running:
```bash
pip install mujoco
```

Output showed:
```
Requirement already satisfied: mujoco in
  /home/air-lab-ncsu/anaconda3/envs/openarm/lib/python3.11/site-packages (3.6.0)
```

MuJoCo 3.6.0 was already present in the `openarm` env.
The earlier `ModuleNotFoundError` was only because the wrong Python was being used.

**Key lesson:** If you get `ModuleNotFoundError` for a package you think is installed,
always verify which Python is active:
```bash
which python       # shows which python executable is being used
conda activate openarm  # make sure you're in the right env
```

---

## Running the Simulation Demo

```bash
cd ~/OPEN_ARM/openarm
python examples/simulation_demo.py
```

**What the demo does:**
- Loads the full OpenArm dual-arm robot model from `openarm/simulation/models/openarm.xml`
- Opens a MuJoCo interactive viewer window
- Runs a sinusoidal oscillation on joint 1 of both arms simultaneously

```python
# Core logic inside simulation_demo.py:
amplitude = 0.5      # radians — how far the joint swings
frequency = 0.01     # how fast it oscillates (cycles per simulation step)
first_joint_angle = amplitude * math.sin(step_count * frequency)
```

Both arms swing their shoulder (joint 1) back and forth in sync.
The other 6 joints hold their last position.

**To stop:** Close the MuJoCo viewer window.

---

## What the MuJoCo Viewer Shows

When the viewer opens you will see:

```
- Dual-arm robot mounted on a base
- Left arm  (7 joints + gripper)
- Right arm (7 joints + gripper)
- Both shoulder joints oscillating smoothly
```

Mouse controls in the viewer:
```
Left click + drag    → rotate the camera
Right click + drag   → pan the camera
Scroll wheel         → zoom in / out
Double-click body    → select and inspect a body part
```

---

## The OpenArm Simulation API (What the Code Can Do)

The `OpenArmSimulation` class (in `openarm/simulation/__init__.py`) gives you:

| Method | What it does |
|---|---|
| `sim.get_left_arm_positions()` | Read all 7 joint angles of left arm (radians) |
| `sim.get_right_arm_positions()` | Read all 7 joint angles of right arm (radians) |
| `sim.set_left_arm_position_control(positions)` | PD-control left arm to target angles |
| `sim.set_right_arm_position_control(positions)` | PD-control right arm to target angles |
| `sim.set_left_arm_torques(torques)` | Apply raw torques to left arm joints (N⋅m) |
| `sim.set_right_arm_torques(torques)` | Apply raw torques to right arm joints (N⋅m) |
| `sim.step()` | Advance physics by one timestep |
| `sim.get_left_gripper_position()` | Read left gripper position (meters) |
| `sim.set_left_gripper_position_control(pos)` | Move left gripper to target position |

---

## Package Structure Explored This Session

```
openarm/
├── examples/
│   └── simulation_demo.py      ← the script we ran
├── openarm/
│   ├── damiao/                 ← real hardware control (CAN bus, motors)
│   ├── kinematics/             ← inverse kinematics (IKPy)
│   │   ├── inverse/ikpy.py
│   │   └── models/openarm.urdf
│   ├── simulation/             ← MuJoCo simulation ← NEW this session
│   │   ├── __init__.py         ← OpenArmSimulation class
│   │   ├── models/
│   │   │   ├── openarm.xml     ← MuJoCo model definition
│   │   │   └── meshes/         ← visual + collision mesh files
│   │   └── README.md
│   ├── bus/
│   ├── netcan/
│   └── utils/
```

---

## Quick Reference

```bash
# Make sure you are in the right environment
conda activate openarm
which python
# → /home/air-lab-ncsu/anaconda3/envs/openarm/bin/python

# Run the simulation demo
cd ~/OPEN_ARM/openarm
python examples/simulation_demo.py

# Verify mujoco is installed
pip show mujoco
# → Version: 3.6.0
```

---

## What Comes Next

```
1. Modify the demo — make more joints move, not just joint 1
2. Try torque control instead of position control in simulation
3. Use the kinematics module (IKPy) to compute joint angles
   from a desired end-effector position (x, y, z)
4. Record trajectories in simulation and replay them
5. Eventually: replay recorded sim trajectories on real hardware
```

---

*Written after Step 2 session with OpenArm at AIR Lab, NC State University.*
