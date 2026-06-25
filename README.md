# cub3D

> A Wolfenstein 3D-inspired raycasting engine developed in C using the [Codam MLX42](https://github.com/codam-coding-college/MLX42) graphics library.

This project implements a complete software-based 3D renderer using classical raycasting techniques, featuring texture mapping, sprite rendering, collision detection, map validation, and several gameplay extensions such as portals, animated enemies and a real-time minimap.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Engine Architecture](#engine-architecture)
- [Parsing System](#parsing-system)
- [Map Validation](#map-validation)
- [Player & Camera System](#player--camera-system)
- [Raycasting & DDA](#raycasting--dda)
- [Texture Mapping](#texture-mapping)
- [Sprite Engine](#sprite-engine)
- [Portal System](#portal-system)
- [Door System](#door-system)
- [Collision System](#collision-system)
- [Minimap](#minimap)
- [Controls](#controls)
- [Build](#build)
- [Performance](#performance)
- [Technologies](#technologies)

---

## Overview

cub3D is a real-time first-person renderer built around the same mathematical principles used by early 1990s engines such as Wolfenstein 3D.

The engine performs ray-casting-based wall rendering via DDA grid traversal, perspective projection, texture mapping, sprite rendering with depth sorting, collision detection, and map validation through flood fill. It is entirely implemented in C and uses MLX42 for window management and pixel drawing.

---

## Features

### Mandatory

| Feature | Status |
|---|---|
| Map parsing | ✅ |
| Texture loading (PNG) | ✅ |
| RGB floor and ceiling colors | ✅ |
| Wall rendering via raycasting | ✅ |
| WASD movement | ✅ |
| Player rotation | ✅ |
| Collision detection | ✅ |
| Flood fill map validation | ✅ |

### Bonus

| Feature | Status |
|---|---|
| Mouse camera control | ✅ |
| Minimap | ✅ |
| Doors | ✅ |
| Portals | ✅ |
| Animated enemies | ✅ |
| Sprite rendering with depth buffer | ✅ |
| Sprint mechanic | ✅ |
| Dynamic point of view (pitch) | ✅ |
| PNG signature validation | ✅ |

---

## Engine Architecture

```
.cub file
    │
    ▼
Parser
    │
    ▼
Map Validation (Flood Fill)
    │
    ▼
Engine Initialization
    │
    ▼
Game Loop
    ├── Input
    ├── Movement & Collision
    ├── Raycasting
    ├── Sprite Rendering
    ├── Minimap
    └── UI Rendering
```

---

## Parsing System

The parser reads and validates a `.cub` scene file defining textures, colors and the map layout.

**Texture directives:**
```
NO textures/north.png
SO textures/south.png
WE textures/west.png
EA textures/east.png
```

**Color directives:**
```
F 220,100,0
C 225,30,0
```

### PNG Signature Validation

Every texture path is validated before engine startup. File existence is checked via `open()` and the first 8 bytes are verified against the canonical PNG signature:

```
89 50 4E 47 0D 0A 1A 0A
```

### RGB Validation

RGB values undergo:

- Comma count verification
- Digit-only verification
- Range verification (0–255)
- Malformed input detection

```
✅  F 220,100,0
❌  F ,,255
❌  F 255,
❌  F 255,,0
```

---

## Map Validation

The map is normalized into a rectangular matrix by padding shorter rows with spaces internally.

```
Input:          Normalized:
111111          111111
1001    →       1001__
111111          111111
```

### Flood Fill

A flood fill algorithm is executed from the player's starting position to verify:

- Map closure (no leaks to the boundary)
- All reachable floor tiles are enclosed
- No invalid entities are reachable

If any open path reaches the map boundary, the engine aborts with:
```
Error: map not closed
```

---

## Player & Camera System

Player coordinates are stored as `double` for smooth sub-cell movement:

```c
double pos_row;
double pos_col;
```

### Direction Vectors

| Direction | `dir_x` | `dir_y` |
|-----------|---------|---------|
| North | 0 | -1 |
| South | 0 | +1 |
| East | +1 | 0 |
| West | -1 | 0 |

### Camera Plane

The projection plane is derived from the direction vector:

```c
plane_x = -direct_y * fov;
plane_y =  direct_x * fov;
```

Default `fov = 0.66`, equivalent to approximately **66°** horizontal field of view.

---

## Raycasting & DDA

One ray is cast per screen column:

```c
for (int x = 0; x < screen_width; x++) { ... }
```

For each ray, the engine computes the camera coordinate, ray direction, and traverses the grid using the **Digital Differential Analyzer** algorithm:

```c
while (!hit)
{
    if (side_dist_x < side_dist_y)
    {
        side_dist_x += delta_dist_x;
        map_x += step_x;
    }
    else
    {
        side_dist_y += delta_dist_y;
        map_y += step_y;
    }
}
```

Complexity: O(cells crossed per ray).

### Fisheye Correction

Perpendicular wall distance (`perp_wall_dist`) is used instead of Euclidean distance to eliminate fisheye distortion. Wall height is then:

```c
line_height = screen_height / perp_wall_dist;
```

---

## Texture Mapping

Textures are loaded via `mlx_load_png()` and copied into custom structs, decoupling the renderer from MLX texture objects:

```c
typedef struct s_tex
{
    uint8_t *pixels;
    int      width;
    int      height;
    int      channels;
} t_tex_bytes;
```

Vertical texture sampling uses `int64_t texpos` with fixed-point arithmetic for improved performance and precision.

---

## Sprite Engine

The bonus renderer includes a complete sprite pipeline:

1. **Project** sprite into camera space
2. **Depth test** against the Z-buffer (`buffer_col[x]` stores `perp_wall_dist` per column)
3. **Sort** sprites by distance (back to front)
4. **Render** column by column, skipping occluded pixels

### Animated Enemies

Enemies use sprite atlases sliced into individual frames via `slice_texture()`. Animation states (Idle, Walk, Attack) are cycled at runtime.

---

## Portal System

Two portals can be placed in the world and interact with the raycasting engine.

| Map Tile | Portal Type |
|----------|-------------|
| `4` | Open Portal |
| `5` | Close Portal |

When the player enters a portal, `teleport()` restores the position, direction vector, and camera plane that were stored at placement time, preserving orientation consistency after teleportation.

**Controls:** `Q` places the open portal, `Z` places the close portal.

---

## Door System

| Map Tile | State |
|----------|-------|
| `3` | Closed |
| `2` | Open |

Doors are toggled by pressing `E` while facing them. Closed doors block both movement and raycasting.

---

## Collision System

Movement is resolved separately on each axis to prevent corner clipping:

```c
if (!is_block(new_pos_x, current_pos_y))
    player.pos_x = new_pos_x;
if (!is_block(current_pos_x, new_pos_y))
    player.pos_y = new_pos_y;
```

Tiles treated as solid:

| Tile | Type |
|------|------|
| `1` | Wall |
| `3` | Closed Door |
| `4` | Open Portal |
| `5` | Close Portal |

---

## Minimap

The minimap is rendered directly into the framebuffer each frame. Each tile occupies **6×6 pixels**.

| Color | Tile |
|-------|------|
| Dark grey | Wall (`1`) |
| Light grey | Floor (`0`) |
| Brown | Door (`3`) |
| Blue | Open Portal (`4`) |
| Orange | Close Portal (`5`) |
| Green | Player |

---

## Controls

| Key | Action |
|-----|--------|
| `W` `A` `S` `D` | Move |
| `←` `→` | Rotate camera |
| Mouse | Camera rotation |
| `Shift` | Sprint |
| `E` | Toggle door |
| `Q` | Place open portal |
| `Z` | Place close portal |
| `C` | Change POV (pitch) |
| `ESC` | Exit |

---

## Build

**Dependencies:** GCC, CMake, GLFW, MLX42, libft

```bash
# Mandatory
make

# With bonus features
make bonus

# Clean
make clean
make fclean

# Rebuild
make re
```

---

## Performance

Key optimizations applied throughout the engine:

- DDA traversal (no trigonometry per ray step)
- Fixed-point arithmetic for texture sampling
- Preloaded texture pixel buffers (no runtime PNG decoding)
- Column-based rendering loop
- Per-column depth buffer for sprite occlusion
- Precomputed sprite animation frames
- Direct framebuffer pixel writes via MLX42

Ray parameters are computed once per column, avoiding expensive per-pixel trigonometric operations during rendering.

---

## Technologies

| Component | Technology |
|-----------|------------|
| Language | C |
| Graphics | [MLX42 (Codam)](https://github.com/codam-coding-college/MLX42) |
| Window Backend | GLFW |
| Build System | CMake + Make |
| Platform | Linux |

---

*This project explores the fundamental techniques behind early software-rendered 3D engines: linear algebra, vector mathematics, camera transformations, DDA grid traversal, texture mapping, collision detection, sprite projection, and real-time game engine architecture — reproducing the core rendering pipeline of classic FPS games while extending it with portals, animated entities and interactive environments.*
