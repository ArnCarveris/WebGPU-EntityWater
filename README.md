# Entity Water // WebGPU

A GPU river simulation over a heightmap in WebGPU, after Filip Strugar's
[RiverSim](https://github.com/fstrugar/riversim). A shallow-water flow solver moves water over the
terrain from springs, rain and tools into rivers, a dammed lake and the sea. Surface-wave cascades
that follow the camera ride on the flow field, and the water is rendered with screen-space
refraction, depth absorption, sky reflection and advected foam.

RiverSim ran in two stages: first an offline flow sim until it settled, then a realtime preview with
waves on top. Here both run live every frame. Water keeps flowing while you breach the dam, dig
channels, pour water or make it rain.

Everything is in one static file, `index.html`: the engine classes, the WGSL, and the scenario as
data. Open it in a WebGPU browser (Chrome/Edge 113+, or Brave with WebGPU enabled), straight from
disk or served over HTTP:

```bash
python -m http.server 8768
```

Drop another scenario `.json` on the page to load it.

## How it works

**Flow sim** (`WGSL_FLOW`, `FlowSim`): 512 × 512 cells of 4 m. The state is one `rgba32float`
texture (depth, velocity in cells per step, wet). Each step runs two compute passes, as in RiverSim:

1. **simulate**:
   - Water moves toward each of the 8 neighbours along the water-surface difference, clamped by the
     water available on each side. The clamp is antisymmetric, so this exchange conserves mass.
   - The same differences accelerate the velocity by `g·dt²/dx`.
   - RiverSim's regulation is applied: speed capped at 0.499 cells per step, linear and quadratic
     damping, and extra friction in shallow water (by the cell's own depth and the minimum depth of
     its 3 × 3 neighbourhood).
   - A CFL cap limits the acceleration in deep water, so lakes stay stable.
   - Sources and sinks are added, and the edge condition is applied (open, closed, or held at sea
     level).
2. **propagate**: each neighbour's water (with its momentum) moves by its velocity and lands on a
   bilinear footprint. This forward advection conserves mass. Velocity is lightly smoothed.

After the steps, an **info** pass writes what rendering needs:

- a surface height texture (extended one cell over dry banks, so the water mesh meets the terrain
  in a smooth shoreline)
- velocity in m/s and turbulence (speed / depth-based friction, as in RiverSim's exporter)
- total volume, wet area and top speed, gathered with workgroup atomics and read back
  asynchronously for the HUD

The sim runs 32 steps of 0.12 s per frame (about 550× real time), plus a warm-up burst after
loading.

**Surface waves** (`WGSL_WAVES`, `WaveCascade`): RiverSim's realtime part. Each cascade layer is a
512² finite-difference wave map centred on the camera: 192 m at 0.38 m per texel, and 768 m at
1.5 m per texel.

- Each step reads the previous state upstream (semi-Lagrangian, by the flow velocity) and from
  where the texel sat before the layer moved.
- Sparse impulses excite it, stronger where the flow is turbulent. Waves are damped, and cut where
  there is no water.
- A foam map is advected the same way. Steep wave fronts and turbulence add foam, and it slowly
  dissolves.
- Every layer steps at `speed / (0.707 · texel)` Hz, so waves travel at the same speed in all
  layers.
- A display pass writes height, gradient and foam into one of two slots. Rendering blends the two
  newest steps, so a 5 Hz layer still moves smoothly.

**Render** (`Renderer`): two passes.

1. **Scene pass**, into an HDR target: the sky, then the terrain.
   - Soft sun shadows are ray-marched through the heightfield.
   - Terrain is coloured by slope and height, with darkened banks where water has recently been
     (the sim's `wet` channel).
   - Riverbeds get caustics.
2. **Surface pass**: the tonemapped scene, then the water mesh, then the debris.
   - Water refracts the scene colour and depth (`depthReadOnly`).
   - Light is absorbed along the refracted ray and scattered with the deep colour.
   - Sky is reflected with Fresnel, and the sun gives a specular glint.
   - Normals combine the surface slope, the wave cascades and two-phase flow-mapped ripples.
   - Foam comes from the wave foam, turbulence and fast shallows.

   Dry water vertices next to water sit just under the terrain, so the shoreline is where the two
   meshes intersect. The other dry vertices sit far below, so early-z drops them.

**Debris**: floating logs, updated by a compute shader.

- Logs drift with the current and turn to follow it.
- A log that strands, or reaches the end of its life, respawns at the deepest of a few random
  spots around its emitter.

## Controls

| Key | Action |
|---|---|
| drag / WASD / Space, C | look / move / up, down |
| Shift, Alt, wheel | ×5, ×0.2, speed |
| right drag (or Ctrl + drag) | use the current tool where the cursor meets the terrain |
| 1–5 | tools from the scenario: pour, splash, raise, dig, drain |
| [ ] | brush radius |
| B | boost springs (×8) |
| R | rain on/off |
| X | breach / rebuild the dam |
| N | reset the water to its initial state |
| M | render mode: shaded, depth, velocity, foam / turbulence, wave cascades |
| V, G | next camera view / next lighting preset |
| , . | sim steps per frame −8 / +8 |
| L, P, H | labels, pause, help |

## Scenario (`<script id="scenario">`)

| Key | Contents |
|---|---|
| `terrain` | `size` (m), `resolution` (cells per side; also the flow grid), `base` height, `snowLine` |
| `sim` | `stepsPerFrame`, `stepTime` (s), `gravity`, `diffusion`, `linDamping`, `sqrDamping`, `absorption`, `evaporation` (mm/h), `frictionMinDepth` / `frictionMaxDepth` / `frictionAmount`, `minDepth`, `wetDecay`, `cfl`, `turbulenceSpeed`, `warmup` (steps), `edges` (`open` / `closed`) |
| `waves` | `speed` (m/s), `damping`, `noise`, `foam`, `layers` [{ `size`, `resolution`, `height` }] |
| `water` | `deep` colour, `absorb` (per m, rgb), `refraction`, `foam`, `detail`, `flowScale` (visual speed of advected detail and debris) |
| `entities` | `{ type, id, label, ... }`, where `type` maps to a class in `ENTITY_TYPES` (below), applied in order |
| `tools` | `{ type, key, rate / strength }`, types `pour`, `drain`, `raise`, `dig`, `splash` |
| `lighting` | named presets plus `start`: `sun` { `azimuth`, `elevation`, `color`, `intensity` }, `sky` { `top`, `horizon` }, `ground`, `ambient`, `fog`, `exposure` |
| `views` | `{ name, pos [x, y, z], look [x, y, z] }` |

Positions are metres: `[x, z]` on the map, with x east, z south, and the map centred on 0.

### Entity types

| Type | Kind | Parameters |
|---|---|---|
| `tilt` | terrain | `dir` (downhill), `drop` (m across the map) |
| `hills` | terrain | fractal noise: `amplitude`, `scale`, `octaves`, `seed`, `ridged` |
| `mountain` | terrain | `pos`, `radius`, `height`, `roughness` (ridged noise) |
| `valley` | terrain | carves a river valley along `path`: `width`, `channel` depth, `bank` width, `levels` [start, end], `wiggle` |
| `basin` | terrain | lake bowl: `pos`, `radius`, `floor`, optional `rim` height and `rimWidth` |
| `coast` | terrain | lowers the map past `at` (along `dir`) to `floor` over `width` |
| `dam` | terrain | wall `from` → `to`: `crest`, `top` width, `side` slope, `spillway` { `width`, `depth` }, `gap` { `at`, `width` }, `breachTime`. X breaches it and rebuilds it. |
| `sea` | water | fills everything below `level` connected to the map edge, and holds the edge at that level |
| `lake` | water | flood-fills from `pos` up to `level` |
| `spring` | water | `pos`, `rate` (m³/s), `radius`, `boost` |
| `rain` | water | `rate` (mm/h), everywhere or over `pos` + `radius`; `enabled` |
| `drain` | water | sink: `pos`, `rate` (m³/s), `radius` |
| `debris` | water | floating logs: `pos`, `radius`, `count`, `size`, `color`, `life` (s) |

### Adding a new kind of entity

1. Subclass `Entity` and override the hooks it needs:
   - `stamp(field)`: shape the heightfield at build time.
   - `fill(water)`: set the initial water depth.
   - `spawn()`: register with the world.
   - `sources(out)`: push `{ x, z, radius, rate, flat }` water sources (negative `rate` drains), or
     add m/s to `out.rain`.
   - `update(dt, t)`: per-frame changes. To edit terrain at runtime, change `world.field.h` and
     call `field.touch(rect)`. Only that rectangle is uploaded.
2. Register it in `ENTITY_TYPES` and add it to the scenario's `entities`.

## Layout (sections of the script in `index.html`)

```
config, math, noise                  constants, vectors / matrices, value noise, fbm, polylines
Heightfield, floodFill               CPU terrain: authoring, sampling, brush, raycast, dirty-rect upload
WGSL_FLOW, WGSL_WAVES, WGSL_DEBRIS_SIM  compute: flow sim (simulate / propagate / info), wave cascades, debris
WGSL_COMMON, WGSL_SCENE, WGSL_SURFACE   render: shared frame + helpers, sky / terrain, composite / water / debris
Entity, ENTITY_TYPES, World          data-driven world
FlowSim, WaveLayer, WaveCascade, DebrisSystem, Renderer
Input, FlyCamera, Tool, TOOL_TYPES, Hud, App, main
```
