# LightingSystem

`LightingSystem` manages a GPU-accelerated GLSL lighting pass (point lights, directional lights, optional normal maps). It is instance-based — create one per scene and call `update(dt)` each frame to upload light positions to the GPU.

## Import

```typescript
import { LightingSystem, type Light, type LightType } from '@emptysock/engine'
```

## Setup (in onLoad)

Lighting requires `lighting: true` in `SceneConfig`:

```typescript
import { Scene, type SceneConfig } from '@emptysock/engine'

export class DungeonScene extends Scene {
  static readonly config: SceneConfig = {
    renderMode: '2d',
    gameSpeed: 60,
    lighting: true,   // required
  }

  private _lighting!: LightingSystem

  override onLoad(): void {
    this._lighting = new LightingSystem()
    // Attach the GPU filter to the scene stage (provided by the engine):
    this._lighting.attachFilter(this.stage)
    this._lighting.setAmbient(0x111133, 0.08)
  }

  override onUpdate(dt: number): void {
    this._lighting.update(dt)   // uploads light data to GPU uniforms each frame
  }
}
```

## Adding lights

Lights are identified by a string `id` you supply. The `Light` object is stored in `lighting.lights` and can be mutated in-place to move lights without remove/re-add:

```typescript
this._lighting.addLight({
  id:          'torch-1',
  type:        'point',
  x:           300,
  y:           200,
  colour:      0xffaa44,
  intensity:   1.4,
  radius:      280,
  castShadows: false,
})

// Directional light:
this._lighting.addLight({
  id:        'sun',
  type:      'directional',
  colour:    0xfffbe6,
  intensity: 0.8,
  direction: { x: 0.5, y: -1 },
  castShadows: false,
})
```

## Moving a light

Mutate the light's `x`/`y` directly — no need to remove and re-add:

```typescript
override onUpdate(dt: number): void {
  const torch = this._lighting.lights.get('torch-1')
  if (torch !== undefined) {
    torch.x = this._playerEntity.position.x
    torch.y = this._playerEntity.position.y - 20
  }
  this._lighting.update(dt)
}
```

## Removing lights

```typescript
this._lighting.removeLight('torch-1') // returns boolean — true if found
```

## Ambient light

```typescript
this._lighting.setAmbient(0x111133, 0.08)

const colour    = this._lighting.ambientColour    // number (hex)
const intensity = this._lighting.ambientIntensity // number
```

## Light interface

```typescript
interface Light {
  readonly id:         string;
  readonly type:       'point' | 'directional' | 'spot' | 'ambient';
  colour:              number;   // 0xRRGGBB — mutable
  intensity:           number;   // 0..1+ — mutable
  radius?:             number;   // point/spot falloff radius in world pixels
  angle?:              number;   // spot: cone half-angle in radians
  direction?:          { x: number; y: number };  // directional: world-space
  castShadows:         boolean;
  x?:                  number;   // world position — mutable
  y?:                  number;   // world position — mutable
}
```

## API reference

| Method / Property | Signature | Notes |
|---|---|---|
| `addLight` | `(config: Light): void` | Register a light by id. Replaces if id already exists. |
| `removeLight` | `(id: string): boolean` | Remove light by id. Returns true if found. |
| `lights` | `Map<string, Light>` (readonly) | All registered lights — mutate to move/recolour without re-adding. |
| `setAmbient` | `(colour: number, intensity: number): void` | Scene-wide ambient. |
| `ambientColour` | `number` (getter) | Current ambient colour. |
| `ambientIntensity` | `number` (getter) | Current ambient intensity. |
| `attachFilter` | `(stage: Container, useNormalMap?: boolean, width?: number, height?: number): void` | Wire GPU filter to the scene stage. Call once in onLoad. |
| `detachFilter` | `(): void` | Detach the GPU filter. Call in onDestroy. |
| `update` | `(dt: number): void` | Upload all light data to GPU uniforms. Call every frame. |

## Notes

- `attachFilter()` must be called before lights affect rendering — without it, lights are registered but not drawn.
- Normal maps: if `hero.png` exists and `hero_n.png` exists beside it, pass `useNormalMap: true` to `attachFilter` and the engine applies it automatically. The normal map must be a tangent-space normal map (blue-dominant).
- Limit shadow-casting lights (`castShadows: true`) — each casts an extra GPU pass.
- On `'potato'` and `'low'` GPU tiers (`Engine.gpuTier`), keep `castShadows: false` and use at most 1–4 point lights.
- Up to 16 point lights and 4 directional lights per scene (GPU uniform array limits).
