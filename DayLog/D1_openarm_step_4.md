# OpenArm Session — Step 4: Dual-Arm Oscillation + Hardware-Simulation Mirror
### Running both arms and watching them move in MuJoCo at the same time
**AIR Lab, NC State University**

---

## What Was Working Before This Step

Single-arm oscillation on `robot_l` confirmed working:

```
(openarm) air-lab-ncsu@...$ python examples/hardware_oscillation_demo.py --port robot_l:left
Enabling 8 motors on robot_l (left arm)...
Initial joint positions (rad): [0.013, -0.008, -0.298, -0.015, -1.159, -0.249, -0.005, -0.274]

Starting oscillation on joint 1 (shoulder rotation).
  Amplitude : 0.3 rad  (~17.2 degrees)
  All other joints hold their starting positions.

Press  Q  to stop.

Stopping — disabling all motors...
Done. All motors disabled.
```

Joint 1 physically swung ±17° while all other joints stayed frozen. ✅

---

## Goal of This Step

1. **Dual-arm oscillation** — run the same motion on both arms simultaneously
2. **Hardware + Simulation mirror** — read actual motor positions from the hardware
   and display them inside MuJoCo so you can see the virtual twin move with the real arm

---

## Part 1: Dual-Arm Oscillation

### What changed in `hardware_oscillation_demo.py`

The `--port` argument now accepts **multiple values** (one per arm):

```bash
# Before (single arm only)
--port robot_l:left

# After (both arms)
--port robot_l:left --port robot_r:right
```

Internally, the script now runs both arms **concurrently** using `asyncio.gather`:

```
asyncio event loop
    ├── keyboard_watcher()     ← watches for Q key
    ├── oscillate_arm(left)    ← sends CAN commands to robot_l at 100 Hz
    └── oscillate_arm(right)   ← sends CAN commands to robot_r at 100 Hz
```

`asyncio.gather` runs all three coroutines at the same time.
The CAN messages interleave between the two arms on each 0.01s loop cycle.

### How to run dual-arm oscillation

```bash
# Make sure both CAN interfaces are up
sudo ip link set robot_l up type can bitrate 1000000
sudo ip link set robot_r up type can bitrate 1000000

# Run both arms
python examples/hardware_oscillation_demo.py --port robot_l:left --port robot_r:right
```

---

## Part 2: Hardware + Simulation Mirror

### What is it?

A new script `examples/hardware_sim_mirror.py` that does three things at once:

```
Real arm (CAN bus)           Shared state              MuJoCo viewer
─────────────────            ────────────              ─────────────
Motor replies               ┌──────────────┐           Viewer reads positions
with actual position   ───► │ left[0..6]   │ ────────► and calls
(e.g. 0.312 rad)            │ right[0..6]  │           set_left_arm_positions()
                            └──────────────┘           viewer.sync()
```

The **real arm oscillates**, the **sim twin follows** — both move in sync.

### Why two threads?

MuJoCo's viewer must run in the **main thread** (OS requirement for GUI windows).
CAN communication uses Python `asyncio` which needs its own event loop.
These two cannot share the same thread, so:

```
Main thread (viewer):
    sim = OpenArmSimulation()
    with mujoco.viewer.launch_passive(...) as viewer:
        while viewer.is_running():
            left, right = shared_state.read()       ← reads from background thread
            sim.set_left_arm_positions(left)
            viewer.sync()

Background thread (CAN):
    asyncio.run(can_main(...))
        → sends MIT commands to hardware
        → reads back actual motor positions
        → shared_state.write_arm("left", actual_positions)   ← writes for main thread
```

### SharedState class

The `SharedState` class is a thread-safe container with a `threading.Lock`:

```python
shared.write_arm("left", actual_positions)   # called by CAN thread
left, right, lgrip, rgrip = shared.read()    # called by viewer thread
```

The lock prevents both threads from reading/writing at the same time,
which would cause corrupted or half-updated position arrays.

### What the MuJoCo simulation shows

The sim uses `set_left_arm_positions()` which sets joint angles **directly**
without running physics. This is different from simulation_demo.py which
uses position control + `sim.step()`.

```
sim.set_left_arm_positions(left)    ← teleports joints to hardware positions
sim.set_left_gripper_position(grip) ← same for gripper
viewer.sync()                       ← refreshes the display
```

Because there's no `sim.step()`, the sim is not simulating physics —
it is purely a **visualizer** of what the real arm is doing.

### How to run the mirror

```bash
# Left arm only
python examples/hardware_sim_mirror.py --port robot_l:left

# Both arms
python examples/hardware_sim_mirror.py --port robot_l:left --port robot_r:right
```

Expected output:
```
[robot_l] Enabling 8 motors (left arm)...
[robot_l] Initial positions: [0.013, -0.008, -0.298, ...]

MuJoCo viewer opening — real arm positions will appear there.
Press  Q  in this terminal to stop.
```

A MuJoCo window opens. The virtual arm copies the real arm's motion live.

### Stop
Press **Q** in the terminal.
The CAN thread disables all motors and shuts down the CAN bus.
The viewer window also closes.
If you close the viewer window first, the script also stops automatically.

---

## Files Created / Modified This Session

| File | What changed |
|---|---|
| `examples/hardware_oscillation_demo.py` | Updated: `--port` now accepts multiple values for dual-arm |
| `examples/hardware_sim_mirror.py` | New: hardware oscillation + live MuJoCo mirror |

---

## Key Concept: asyncio vs threading

| | asyncio | threading |
|---|---|---|
| What it is | Cooperative multitasking in one thread | Multiple OS threads running in parallel |
| Used for | CAN communication (I/O-bound) | MuJoCo viewer (must be main thread) |
| How tasks run | Take turns voluntarily (`await`) | True concurrency (GIL limited in Python) |
| How they talk | `SharedState` with a `threading.Lock` | Same |

The `await asyncio.sleep(0.01)` in the CAN loop is the moment where other
async tasks (like `keyboard_watcher`) get their turn to run.

---

## What Comes Next

```
1. Instead of only oscillating joint 1, move the arm by hand
   (gravity compensation mode) and watch the sim mirror your motion live

2. Record the positions from hardware during a hand-guided motion
   → save to a CSV or numpy array

3. Replay that recorded trajectory on the hardware
   → the arm repeats your motion automatically

4. Play the trajectory in simulation first to check it looks right,
   then send to hardware
```

---

*Written after Step 4 session — dual-arm and hardware/sim mirror at AIR Lab, NC State University.*
