# LightingSystem

**Use this when** you want dynamic 2D lighting (torches, a day/night cycle, cones, shadows from walls) instead of baked-in lighting on your sprites. Lights are entities: attach a `LightSource` component next to a `Transform`, and optionally `LightOccluder` components to block light. `LightingSystem` collects them each frame (with real occlusion-aware visibility polygons) and, once attached with `renderPipeline.attachLighting(lighting)`, `RenderPipeline.renderFrame()` turns the result into a lightmap multiplied over a render layer.

## Import

```typescript
import {
  LightingSystem, LightSource, LightOccluder, Transform,
  type AmbientLight, type LightSample, type LightingSystemOptions,
} from '@emptysock/engine'
```

## Setup

```typescript
const lighting = new LightingSystem({ maxLights: 32, raySamples: 32 })   // both optional (defaults 32 / 32)
lighting.ambient = { colour: 0x111133, level: 0.08 }   // 0 = pitch black except lit areas, 1 = fully lit
```

`ambient` is a plain writable property (`{ colour: 0xRRGGBB, level: 0..1 }`); there is no `setAmbient()`.

## Adding lights

A light is an entity with `Transform` + `LightSource`:

```typescript
const torch = scene.spawn('torch')
torch.add(Transform, { x: 300, y: 200 })
torch.add(LightSource, {
  radius: 280,
  colour: 0xffaa44,
  intensity: 1.4,
  falloff: 1,
})

// A spotlight cone (angle and direction in degrees):
const lamp = scene.spawn('lamp')
lamp.add(Transform, { x: 600, y: 100 })
lamp.add(LightSource, { radius: 400, coneAngle: 60, coneDirection: 90 })
```

`LightSource` fields (defaults): `radius` (200), `colour` (0xffffff), `intensity` (1), `falloff` (1), `offsetX` (0), `offsetY` (0), `enabled` (true), `coneAngle` (360 = full point light), `coneDirection` (0). There are no `directional` or `spot` light types, no light `id`, and no `castShadows` flag: shadows come from `LightOccluder`s.

## Moving and removing lights

Mutate the entity's `Transform`, or the `LightSource` fields; destroy the entity to remove the light.

```typescript
const t = torch.get(Transform)
if (t !== undefined) { t.x = playerX; t.y = playerY - 20 }

const l = torch.get(LightSource)
if (l !== undefined) l.enabled = false

scene.destroy(torch)
```

## Shadows: LightOccluder

Add `LightOccluder` (with `Transform`) to a wall or crate entity. It is an axis-aligned box (`width`, `height`, `offsetX`, `offsetY`, `enabled`) that blocks light from any `LightSource` whose radius reaches it. A scene with no occluders renders plain circular falloff.

```typescript
const wall = scene.spawn('wall')
wall.add(Transform, { x: 400, y: 200 })
wall.add(LightOccluder, { width: 64, height: 64 })
```

## Collecting lights

```typescript
const samples: LightSample[] = lighting.collectLights(scene, { x: cameraX, y: cameraY })
// LightSample: { x, y, radius, colour, intensity, falloff, coneAngle, coneDirection (radians), visibility: Point[] | null }
```

Only enabled lights with `radius > 0` are collected. When more than `maxLights` exist, the `maxLights` nearest the reference point win.

## Rendering

Attach the lighting system to the pipeline once and `RenderPipeline.renderFrame()` does the rest every frame: it builds a lightmap for the main scene over the camera's visible world rect and multiplies it over one render layer.

```typescript
renderPipeline.attachLighting(lighting)             // layer defaults to 'default'
renderPipeline.attachLighting(lighting, 'world')    // or pick the layer the darkness applies to
renderPipeline.attachLighting(null)                 // detach: removes the filter and frees the lightmap
```

The camera rect is computed from the stage's translate and scale, so camera rotation is ignored by the lightmap. Only the main scene's lights are collected (overlay scenes are not). Under the hood this calls `RenderSystem.syncLighting(lighting, scene, layerId, viewport)`; `RenderSystem` is exported from `@emptysock/engine` too, but you only need it directly if you build your own render loop without `RenderPipeline`.

## API reference

| Member | Signature | Notes |
|---|---|---|
| `new LightingSystem(options?)` | `{ maxLights?, raySamples? }` | |
| `ambient` | `AmbientLight` (`{ colour, level }`) | Writable property |
| `maxLights` / `raySamples` | `number` | Writable |
| `RenderPipeline.attachLighting` | `(lighting: LightingSystem \| null, layerId = 'default'): void` | Syncs the lightmap every `renderFrame()`; `null` detaches |
| `collectLights` | `(scene: Scene, reference?: { x, y }): LightSample[]` | Enabled lights resolved to world space, capped at `maxLights` |

## Notes

- Occluded lights cost `lights x occluders x rays`; keep occluder counts modest (dozens of walls, 5-10 lights is the intended scale) and lower `raySamples` on weak GPUs.
- On `'potato'` and `'low'` GPU tiers, lower `maxLights` and `raySamples` (see `skills/24-viewport-system.md`).
- Normal maps are not supported; lighting is a lightmap multiplied over the layer.
