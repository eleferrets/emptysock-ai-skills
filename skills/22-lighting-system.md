# LightingSystem

`LightingSystem` manages dynamic lights in a scene. Enable lighting in the scene config first, then use `LightingSystem` to add, configure, and remove lights at runtime.

## Import

```typescript
import { LightingSystem, type Light } from '@emptysock/engine'
```

## Setup

Lighting requires `lighting: true` in `SceneConfig`:

```typescript
import { Scene, type SceneConfig } from '@emptysock/engine'

export class DungeonScene extends Scene {
  static readonly config: SceneConfig = {
    renderMode: '2d',
    gameSpeed: 60,
    lighting: true,   // required — lighting is off by default
  }
}
```

## Ambient light

```typescript
// Set the base ambient colour and intensity (applied to everything not lit):
LightingSystem.setAmbient(0x111133, 0.08)

// Read current values:
const colour    = LightingSystem.ambientColour    // number (hex)
const intensity = LightingSystem.ambientIntensity // number
```

## Adding lights

```typescript
// Add a point light and get its assigned id:
const torchId = LightingSystem.addLight({
  type:        'point',
  x:           300,
  y:           200,
  colour:      0xffaa44,
  intensity:   1.4,
  radius:      280,
  castShadows: true,
})

// Add a spotlight:
const spotId = LightingSystem.addLight({
  type:      'spot',
  x:         400,
  y:         100,
  colour:    0xffffff,
  intensity: 1.0,
  radius:    350,
  angle:     45,           // cone width in degrees
  direction: Math.PI / 2, // radians — pointing downward
})
```

## Moving a light

There is no update method — remove and re-add with new position:

```typescript
LightingSystem.removeLight(torchId)
const newId = LightingSystem.addLight({ type: 'point', x: newX, y: newY, colour: 0xffaa44, intensity: 1.4, radius: 280 })
```

For lights that follow an entity, remove and re-add each frame in `onUpdate`:

```typescript
private _torchId: string | null = null

override onUpdate(dt: number): void {
  if (this._torchId !== null) LightingSystem.removeLight(this._torchId)
  this._torchId = LightingSystem.addLight({
    type:      'point',
    x:         this.entity.position.x,
    y:         this.entity.position.y - 20,
    colour:    0xffaa44,
    intensity: 1.4,
    radius:    280,
  })
}
```

## Removing lights

```typescript
LightingSystem.removeLight(torchId)
```

## Light type reference

```typescript
// Light shape — all fields except type, x, y are optional:
{
  type:        'point' | 'spot' | 'directional',
  x:           number,
  y:           number,
  colour?:     number,   // hex, default 0xffffff
  intensity?:  number,   // default 1.0
  radius?:     number,   // influence radius in pixels
  angle?:      number,   // spot cone angle in degrees
  direction?:  number,   // spot direction in radians
  castShadows?: boolean, // default false
}
```

## API reference

| Method / Property | Signature | Notes |
|---|---|---|
| `LightingSystem.addLight` | `(config: Partial<Light> & { type: string, x: number, y: number }): string` | Returns the light's assigned id. |
| `LightingSystem.removeLight` | `(id: string): void` | Remove a light by id. |
| `LightingSystem.setAmbient` | `(colour: number, intensity: number): void` | Set scene-wide ambient light. |
| `LightingSystem.ambientColour` | `number` (getter) | Current ambient colour. |
| `LightingSystem.ambientIntensity` | `number` (getter) | Current ambient intensity. |

## Notes

- Lighting has a per-scene GPU cost. Limit shadow-casting lights: `castShadows: true` is expensive.
- On `'potato'` and `'low'` GPU tiers (`Engine.gpuTier`), disable `castShadows` and cap dynamic lights at 1–4.
- `LightingSystem` is static — it applies to the currently active scene's lighting pipeline.
- Normal maps are resolved automatically: if `hero.png` is a sprite texture and `hero_n.png` exists alongside it, it is used as the normal map for dynamic lighting without any extra code.
