# OpenArm Robot — Beginner's Guide
### What I Did, What It Means, and How to Do It Again
**AIR Lab, NC State University**

---

## What Is OpenArm?

OpenArm is an open-source robot arm — meaning anyone can build it, use it, and improve it for free. You are working with a **dual-arm setup** — two robot arms, a left one and a right one, each with **8 motors** (7 joints + 1 gripper).

Think of each arm like a human arm:

```
Motor 1  →  Shoulder rotation      (like turning your shoulder in/out)
Motor 2  →  Shoulder flex          (like raising your arm up)
Motor 3  →  Upper arm rotation     (like twisting your upper arm)
Motor 4  →  Elbow flex             (like bending your elbow)
Motor 5  →  Forearm rotation       (like turning your palm up/down)
Motor 6  →  Wrist flex             (like bending your wrist)
Motor 7  →  Wrist rotation         (like tilting your wrist sideways)
Motor 8  →  Gripper                (open and close the hand)
```

**7 joints = 7 Degrees of Freedom (7-DOF).** The gripper is not a DOF because it does not change where the arm can reach — it just opens and closes.

---

## How Does the Robot Actually Work?

Here is the full chain from your laptop to the motors:

```
Your Python command
        |
        v
Linux Computer (Ubuntu)
        |
        v
CAN Interface (robot_l or robot_r)
   — this is like a highway for motor commands —
        |
        v
USB-CAN Adapter (the small box plugged into your laptop)
        |
        v
CAN Bus wire (CANH + CANL + GND)
        |
        v
Motors (Damiao DM4310)
   — motor moves, sends back position/temperature data —
```

**CAN** stands for Controller Area Network. It is a communication protocol originally invented for cars so many devices can talk on one wire. Your robot uses the same technology.

---

## What Is a CAN ID?

Every motor on the CAN bus has a unique ID number — like a house number on a street. When you send a command, you address it to a specific motor by its ID.

Your motors use **two ID numbers**:

| Name | What it is | Example |
|---|---|---|
| `slave_id` | The motor's own address | 1, 2, 3 ... 8 |
| `master_id` | The address motors reply to | slave_id + 16 |

**The formula you discovered today:**
```
master_id = slave_id + 16

Motor 1 → talk to ID 1,  reply comes back on ID 17
Motor 2 → talk to ID 2,  reply comes back on ID 18
Motor 3 → talk to ID 3,  reply comes back on ID 19
...and so on up to Motor 8 → ID 8, reply on ID 24
```

You found this by using `candump` — a tool that lets you watch all CAN messages on the wire like a security camera. You saw:
```
robot_l  001  [8]  FF FF FF FF FF FF FF FC   ← your command going OUT to motor 1
robot_l  011  [8]  11 7F F4 80 08 00 19 15   ← motor 1 replying back (0x011 = 17)
```

---

## What Does the LED Color Mean?

Each Damiao motor has an LED light on it:

| LED Color | Meaning |
|---|---|
| 🔴 Red | Motor is in fault/error state, or was never enabled |
| 🟢 Green | Motor is enabled and healthy, receiving commands |

When you first power on the robot, all LEDs are **red** — the motors are alive but waiting to be told what to do. After you run the `enable` command, they turn **green**.

---

## What is `candump`?

`candump` is like a wiretap on the CAN bus. It prints every single message that travels on the wire in real time.

```bash
candump robot_l
```

Example output explained:
```
robot_l  001   [8]  FF FF FF FF FF FF FF FC
   |      |     |    |
   |      |     |    The actual data bytes (in hexadecimal)
   |      |     Length of message (8 bytes)
   |      CAN ID of who sent it (motor 1 = 0x001)
   Which CAN interface
```

---

## The Commands You Used Today

### Step 1 — Bring up the CAN interfaces

Before talking to the motors, you must turn on the CAN network interface on your laptop. Think of it like turning on Wi-Fi before browsing the internet.

```bash
sudo ip link set robot_l down
sudo ip link set robot_l up type can bitrate 1000000
sudo ip link set robot_r down
sudo ip link set robot_r up type can bitrate 1000000
```

**What `bitrate 1000000` means:** The motors communicate at 1,000,000 bits per second (1 Mbps). Both sides must use the same speed or they cannot understand each other — like two people trying to talk at different speeds.

---

### Step 2 — Enable all motors

```bash
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1 17
```

Breaking this command down:

| Part | Meaning |
|---|---|
| `python -m openarm.damiao` | Run the OpenArm motor control module |
| `enable` | Tell the motor to wake up and accept commands |
| `--motor-type DM4310` | Which model of Damiao motor it is |
| `--iface robot_l` | Which CAN interface (left arm) |
| `1` | slave_id — talk to motor number 1 |
| `17` | master_id — motor should reply to address 17 |

When it works, you see:
```
Motor 1 enabled successfully
Motor 1 state:
  Position: -0.014305 rad     ← where the joint is right now
  Velocity: -0.007326 rad/s   ← how fast it is moving (basically zero = still)
  Torque:   -0.002442 Nm      ← how much force it is applying
  MOS Temp: 25°C              ← electronics temperature
  Rotor Temp: 21°C            ← motor winding temperature
```

**What is `rad`?** Radians — a way to measure angles.
- 0 rad = straight/home position
- 3.14 rad (π) = 180 degrees
- 6.28 rad (2π) = full 360 degree rotation

---

### Step 3 — Gravity Compensation Mode

```bash
python -m openarm.damiao.gravity --port robot_l:left
```

**What this does:** The software calculates how hard gravity is pulling on every part of the arm based on where each joint is. Then it commands each motor to push back with just enough force to cancel gravity out.

**What you feel:** The arm becomes weightless. You can move it with one finger. When you let go, it stays exactly where you left it — floating.

This is not the arm moving by itself. It is the arm becoming neutrally buoyant in gravity — like holding something underwater.

**To stop it:** In a new terminal window, run:
```bash
kill $(pgrep -f "openarm.damiao.gravity")
```

⚠️ **Always hold the arm before stopping gravity mode.** When the process stops, motors return to hold-last-position mode. The arm will not fall, but be careful.

---

### Step 4 — Disable all motors

```bash
python -m openarm.damiao disable --motor-type DM4310 --iface robot_l 1 17
```

Always disable in **reverse order** (8 → 1) so the wrist and gripper go limp first, then elbow, then shoulder. This prevents the arm from jerking dangerously if the shoulder loses power while the rest is still active.

---

## Your Complete Scripts

### Enable all motors (both arms)

```bash
# Run this every time you start a session
for id in 1 2 3 4 5 6 7 8; do
  master=$((id + 16))
  echo "--- Enabling motor $id (master $master) ---"
  python -m openarm.damiao enable --motor-type DM4310 --iface robot_l $id $master
  python -m openarm.damiao enable --motor-type DM4310 --iface robot_r $id $master
  sleep 5
done
echo "All motors enabled!"
```

### Disable all motors (both arms)

```bash
# HOLD BOTH ARMS before running this!
for id in 8 7 6 5 4 3 2 1; do
  master=$((id + 16))
  echo "--- Disabling motor $id (master $master) ---"
  python -m openarm.damiao disable --motor-type DM4310 --iface robot_l $id $master
  python -m openarm.damiao disable --motor-type DM4310 --iface robot_r $id $master
  sleep 5
done
echo "All motors disabled!"
```

---

## What Are Temperatures Telling You?

| Temperature | What it is | Safe range | Danger zone |
|---|---|---|---|
| MOS Temp | The electronics/circuit board | 20–50°C | Above 80°C |
| Rotor Temp | The spinning motor winding | 20–50°C | Above 80°C |

Your motors today ran at 24–27°C — completely cold and healthy. You could run them for hours without any thermal issue.

---

## Important Safety Rules

1. **Always hold the arm** before disabling motors — it will go limp.
2. **Always bring up CAN interfaces** before enabling motors, or commands fail silently.
3. **Never yank a CAN cable** while motors are enabled — the arm may jerk.
4. **Enable shoulder (motor 1) first** — it supports everything above it.
5. **Disable gripper (motor 8) first** — safest to lose wrist before shoulder.
6. **Check temperatures** — if above 70°C, stop and let the arm cool down.

---

## What You Accomplished Today

| Step | What happened |
|---|---|
| Found the USB-CAN adapters | Switched USB port, `lsusb` showed `1d50:606f` |
| Discovered master_id formula | `candump` showed reply on 0x011 → master = slave + 16 |
| Enabled all 8 motors per arm | All LEDs turned green |
| Ran gravity compensation | Arm felt weightless, moved freely |
| Mapped both arms completely | Found 8 motors per arm, not 6 |
| Built enable/disable scripts | Can repeat everything in one command |

---

## What Comes Next

```
1. Dual-arm gravity compensation
   Both arms float at the same time
   Command: python -m openarm.damiao.gravity
              --port robot_l:left --port robot_r:right

2. Teleoperation (leader-follower)
   You move the left arm by hand
   The right arm copies every movement in real time
   This is what --port leader_l:left was designed for

3. Record and replay
   Record your arm movements as a trajectory
   Play it back — the arm repeats your motion automatically

4. Bimanual coordination
   Both arms working together on a task
   The core research goal of OpenArm
```

---

## Glossary

| Word | Simple meaning |
|---|---|
| CAN bus | A wire highway that connects all motors to your computer |
| slave_id | The motor's own address number |
| master_id | Where the motor sends its replies (slave + 16) |
| Bitrate | Communication speed — must be 1,000,000 on this robot |
| DOF | Degree of Freedom — one independent direction of movement |
| Gravity compensation | Software that makes the arm feel weightless |
| Teleoperation | Controlling the robot arm with your own hand movements |
| candump | Tool to watch all CAN messages in real time |
| rad (radian) | Unit for measuring angles (π rad = 180°) |
| Torque | Rotational force the motor is applying (in Newton-meters) |
| MOS Temp | Temperature of the motor's electronic circuit board |
| Rotor Temp | Temperature of the spinning part inside the motor |

---

*Written after first successful session with OpenArm dual-arm setup at AIR Lab, NC State University.*
