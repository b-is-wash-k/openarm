# D2 — Simulation Scene Setup & MuJoCo Background Guide

**Date:** 2026-03-27

---

## What Was Done

### Problem
`simulation_demo.py` was rendering a black/empty viewer because it was:
- Hardcoding a path to an external G1 robot XML (`scene_23dof.xml`) that belonged to a different project
- Not loading the scene file (`scene_openarm.xml`) that provides floor, lighting, and sky
- `scene_openarm.xml` was including `g1_23dof.xml` instead of `openarm.xml`

---

### Files Changed

#### 1. [`openarm/simulation/models/scene_openarm.xml`](../openarm/simulation/models/scene_openarm.xml)

**Before:**
```xml
<mujoco model="g1_23dof scene">
  <include file="g1_23dof.xml"/>
```

**After:**
```xml
<mujoco model="openarm scene">
  <include file="openarm.xml"/>
```

This file is a copy of the G1 23-DOF scene structure. It provides:
- White skybox background
- Groundplane with checker texture
- Directional lighting
- Camera azimuth/elevation defaults

---

#### 2. [`openarm/simulation/models/__init__.py`](../openarm/simulation/models/__init__.py)

Added `OPENARM_SCENE_PATH` so scripts can import the scene path cleanly without hardcoding strings.

**Added:**
```python
OPENARM_SCENE_PATH = Path(__file__).parent / "scene_openarm.xml"

__all__ = ["OPENARM_MODEL_PATH", "OPENARM_SCENE_PATH"]
```

---

#### 3. [`examples/simulation_demo.py`](../examples/simulation_demo.py)

**Before:**
```python
from openarm.simulation import OpenArmSimulation

parser.add_argument("--xml", type=str,
                    default="/home/air-lab-ncsu/AIR_LAB/.../scene_23dof.xml", ...)
sim = OpenArmSimulation()
```

**After:**
```python
from openarm.simulation import OpenArmSimulation
from openarm.simulation.models import OPENARM_SCENE_PATH

parser.add_argument("--xml", type=str, default=str(OPENARM_SCENE_PATH), ...)
sim = OpenArmSimulation(model_path=args.xml)
```

Now the demo loads the scene file by default, giving the robot a proper environment.

---

## How to Change the MuJoCo Background

All background/visual settings live in [`openarm/simulation/models/scene_openarm.xml`](../openarm/simulation/models/scene_openarm.xml) inside the `<asset>` and `<visual>` blocks.

---

### Option 1 — Solid Color Sky

```xml
<asset>
  <texture name="my_sky" type="skybox" builtin="flat"
           rgb1="0.2 0.6 1.0" rgb2="0.2 0.6 1.0"
           width="512" height="512"/>
</asset>
```
Set `rgb1` and `rgb2` to the same color for a uniform solid background.

---

### Option 2 — Gradient Sky (top to bottom)

```xml
<asset>
  <texture name="my_sky" type="skybox" builtin="gradient"
           rgb1="0.3 0.5 0.7"
           rgb2="1.0 1.0 1.0"
           width="512" height="3072"/>
</asset>
```
`rgb1` = top color, `rgb2` = bottom color. The tall height (3072) improves gradient smoothness.

---

### Option 3 — Load an HDR/Panorama Image

```xml
<asset>
  <texture name="my_sky" type="skybox" file="path/to/sky.png"
           width="512" height="512"/>
</asset>
```
Drop a panoramic image into the `meshes/` folder (or provide an absolute path) and reference it here.

---

### Option 4 — Change the Haze/Fog Color

Inside `<visual>`:
```xml
<visual>
  <rgba haze="0.15 0.25 0.35 1"/>   <!-- blue-ish fog -->
</visual>
```
Currently set to `1 1 1 1` (white/no fog). Lower alpha = more fog.

---

### Option 5 — Change the Ground Texture

```xml
<asset>
  <texture type="2d" name="groundplane" builtin="checker"
           rgb1="0.8 0.8 0.8"    <!-- light square color -->
           rgb2="0.4 0.4 0.4"    <!-- dark square color -->
           markrgb="1 1 1"
           width="300" height="300"/>
  <material name="groundplane" texture="groundplane"
            texuniform="true" texrepeat="5 5" reflectance="0.2"/>
</asset>
```
Change `rgb1`/`rgb2` to adjust checker colors, or set `reflectance` higher (0–1) for a shiny floor.

---

### Quick Color Reference

| Color name  | RGB values        |
|-------------|-------------------|
| White       | `1 1 1`           |
| Black       | `0 0 0`           |
| Sky blue    | `0.53 0.81 0.98`  |
| Dark grey   | `0.2 0.2 0.2`     |
| Warm beige  | `0.9 0.85 0.75`   |
| Green field | `0.3 0.6 0.3`     |

---

### Full Example — Dark Studio Look

```xml
<visual>
  <headlight diffuse="0.8 0.8 0.8" ambient="0.1 0.1 0.1" specular="0.2 0.2 0.2"/>
  <rgba haze="0 0 0 1"/>
  <global azimuth="-130" elevation="-20"/>
</visual>

<asset>
  <texture name="dark_sky" type="skybox" builtin="flat"
           rgb1="0.05 0.05 0.05" rgb2="0.05 0.05 0.05"
           width="512" height="512"/>
  <texture type="2d" name="groundplane" builtin="checker"
           rgb1="0.15 0.15 0.15" rgb2="0.1 0.1 0.1"
           markrgb="0.3 0.3 0.3" width="300" height="300"/>
  <material name="groundplane" texture="groundplane"
            texuniform="true" texrepeat="5 5" reflectance="0.05"/>
</asset>
```

---

*All changes apply to the scene file only — no Python code needs to change.*
