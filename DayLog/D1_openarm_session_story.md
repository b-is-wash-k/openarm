# My First OpenArm Session — What Actually Happened
### A beginner's honest account of every command, error, and fix

---

## Part 1 — What is CAN?

**CAN = Controller Area Network.**
Originally invented for cars so many devices (brakes, engine, dashboard) can all talk on one shared wire without needing separate cables for each.

Your robot uses the same idea:

```
All 8 motors share ONE wire pair:
  CANH  (CAN High)
  CANL  (CAN Low)
  GND   (Ground — shared reference)

Your laptop talks to all motors through this wire via a USB-CAN adapter.
```

**How does one motor know a message is for it?**
Every motor has a unique ID number (like a house address). Your laptop says:
> "Hey motor number 1, enable yourself!"

Motor 1 hears it, acts on it. Motors 2–8 hear it too but ignore it because it wasn't addressed to them.

**How does communication actually work?**

```
You type Python command
        ↓
Linux creates a CAN message with motor's ID
        ↓
Message travels through USB into the USB-CAN adapter
        ↓
Adapter converts USB signal → electrical CAN signal
        ↓
Signal travels down CANH/CANL wire to all motors
        ↓
The motor with matching ID reads and responds
        ↓
Response travels back the same wire
        ↓
Your terminal prints motor state (position, temp, torque)
```

---

## Part 2 — The USB-CAN Adapter and Wiring

The USB-CAN adapter is the small box between your laptop and the motors. It has a screw terminal with three connections:

```
USB-CAN adapter          Motor
─────────────            ─────
   CANH        ────→     CANH
   CANL        ────→     CANL
   GND         ────→     GND   ← easy to forget, causes silent failures
```

⚠️ Without GND connected, CAN signals float and communication is unreliable.

---

## Part 3 — How the CAN Interface Got Named `robot_l` and `robot_r`

When you plug a USB-CAN adapter into Linux, it creates a generic interface called `can0`, `can1` etc. The order changes every reboot — bad for a robot.

You ran a setup script that fixed this permanently:

```bash
sudo bash scripts/setup_can.sh leader_l leader_r follower_l follower_r
```

**What the script did internally:**

```
1. Waited for you to plug in the USB-CAN adapter
2. Detected it appeared as can0
3. Read its unique serial number:
      udevadm info -a /sys/class/net/can0
      → ATTRS{serial}=="A50285BI"
4. Wrote a permanent rule to a file:
      /etc/udev/rules.d/90-can.rules
      SUBSYSTEM=="net", ATTRS{serial}=="A50285BI", NAME="robot_l"
5. Now every time that adapter is plugged in,
   Linux renames it robot_l automatically
6. Also created a systemd config to set bitrate:
      /etc/systemd/network/robot_l.network
      [CAN]
      BitRate=1000000
```

**Result:** Instead of random `can0`, `can1` — you always get `robot_l` and `robot_r` no matter what order you plug them in.

---

## Part 4 — Every Error You Hit and What It Meant

### Error 1 — Wrong module path
```
python -m openarm.damiao gravity --port robot_l:left
error: invalid choice: 'gravity'
```
**Why:** `gravity` is its own separate module, not a subcommand.
**Fix:**
```bash
python -m openarm.damiao.gravity --port robot_l:left
#                        ↑ dot here, not a space
```

---

### Error 2 — Missing master_id argument
```
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1
error: the following arguments are required: master_id
```
**Why:** The command needs two numbers — slave_id AND master_id.
**Fix:**
```bash
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1 0
#                                                                    ↑ ↑
#                                                             slave  master
```

---

### Error 3 — Motor got nothing back (`NoneType` crash)
```
AttributeError: 'NoneType' object has no attribute 'data'
```
**Why:** The software sent the enable command, waited for the motor's reply, received NOTHING (`None`), then crashed trying to read data from nothing.

**How you diagnosed it:**
```bash
candump robot_l &    # watch the wire in background
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1 0
```

candump showed:
```
robot_l  001  [8]  FF FF FF FF FF FF FF FC   ← command went OUT ✅
robot_l  011  [8]  11 7F F4 80 08 00 19 15   ← motor replied on ID 0x011 ✅
```

The motor WAS replying — but on ID **17** (0x11 in hex), not ID 0. The software was listening on the wrong address.

**Fix:** master_id must be `slave_id + 16`:
```bash
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1 17
# Motor 1 enabled successfully ✅
```

---

### Error 4 — CAN interface not detected at all
```
ip link   →  no can0, no robot_l, nothing
udevadm info /sys/class/net/can0  →  Unknown device
```
**Why:** The USB-CAN adapter was plugged into a USB-C power-only port that doesn't carry data.

**Fix:** Switched to a proper USB data port. Then:
```bash
lsusb
# Bus 003 Device 011: ID 1d50:606f OpenMoko GS_USB CAN adapter ✅
```

---

### Error 5 — CAN network went down mid-session
```
Error receiving: Network is down [Error Code 100]
Failed to transmit: No such device or address [Error Code 6]
```
**Why:** The `robot_l` interface dropped (possibly after unplugging).
**Fix:** Bring it back up:
```bash
sudo ip link set robot_l down
sudo ip link set robot_l up type can bitrate 1000000
```

---

### Error 6 — Typing normal sentences into terminal
```
no the below 2 are still red
no: command not found

this is also red
Command 'this' not found

now all of the robot (three upper one turns red)
bash: syntax error near unexpected token `('
```
**Why:** The terminal is a command interpreter. It tried to run your English words as programs. Parentheses `()` have special meaning in bash — that's why it gave a syntax error.

**Fix:** Use `echo` to print notes:
```bash
echo "motor 2 is still red"
```

---

## Part 5 — What `candump` Was Showing You

```
robot_l  001   [8]  FF FF FF FF FF FF FF FC
   ↑      ↑    ↑    ↑
   │      │    │    Data bytes in hexadecimal
   │      │    Message length (always 8 bytes)
   │      CAN ID (001 = motor 1)
   Interface name
```

```
FF FF FF FF FF FF FF FC  =  enable command
FF FF FF FF FF FF FF FD  =  disable command
```

When a motor replies, its ID is `slave_id + 16`:
```
robot_l  011  [8]  11 7F F4 ...   ← motor 1 (0x01) replying on 0x11 (=17)
robot_l  012  [8]  12 80 2F ...   ← motor 2 (0x02) replying on 0x12 (=18)
```

---

## Part 6 — The Motor ID Discovery (The Real Breakthrough)

Before this session, nobody had documented the master_id for these motors. You found it live by watching the wire:

```
Sent command  →  to ID 0x001 (motor 1)
Got reply     ←  from ID 0x011 (= 17 in decimal)

0x011 - 0x001 = 0x010 = 16

Therefore:  master_id = slave_id + 16  for ALL motors
```

Complete map for your 8-motor arms:

| Motor | slave_id | master_id |
|---|---|---|
| 1 (shoulder rotation) | 1 | 17 |
| 2 (shoulder flex) | 2 | 18 |
| 3 (upper arm rotation) | 3 | 19 |
| 4 (elbow flex) | 4 | 20 |
| 5 (forearm rotation) | 5 | 21 |
| 6 (wrist flex) | 6 | 22 |
| 7 (wrist rotation) | 7 | 23 |
| 8 (gripper) | 8 | 24 |

---

## Part 7 — Quick Command Reference

```bash
# Check what's plugged into USB
lsusb

# Check CAN interfaces
ip link

# Watch raw CAN messages
candump robot_l

# Bring up CAN interface
sudo ip link set robot_l down
sudo ip link set robot_l up type can bitrate 1000000

# Enable one motor
python -m openarm.damiao enable --motor-type DM4310 --iface robot_l 1 17

# Disable one motor
python -m openarm.damiao disable --motor-type DM4310 --iface robot_l 1 17

# Run gravity compensation
python -m openarm.damiao.gravity --port robot_l:left

# Kill gravity compensation
kill $(pgrep -f "openarm.damiao.gravity")
```

---

*This document covers the actual terminal session at AIR Lab, NC State University — every real error and every real fix.*
