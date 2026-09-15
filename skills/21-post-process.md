# PostProcessSystem

`PostProcessSystem` manages full-screen post-processing effects and per-layer CSS filters. Create one instance per scene; call `update(dt)` each frame for transient effects to decay correctly.

## Import

```typescript
import {
  PostProcessSystem,
  type PostEffectType,
  type PostEffectOptions,
  type LayerFilterOptions,
} from '@emptysock/engine'
```

## Setup (in onLoad)

```typescript
private _post: PostProcessSystem | null = null

override onLoad(): void {
  this._post = new PostProcessSystem()
}

override onUpdate(dt: number): void {
  this._post?.update(dt)   // drives transient effect (flash, shockwave) decay
}

override onDestroy(): void {
  this._post?.destroy()
}
```

## Full-screen effects

`add()` returns `this` for method chaining:

```typescript
// Add effects — composited in the order they were added:
this._post
  .add('bloom', { threshold: 0.7, strength: 0.4, radius: 1.0 })
  .add('vignette', { intensity: 0.45 })

// Check / remove:
if (this._post.has('bloom')) {
  this._post.remove('bloom')
}

// Read the current options for an active effect:
const bloomEffect = this._post.get('bloom') // ActiveEffect | undefined
```

## Transient flash

```typescript
this._post.flash({ colour: 0xffffff, duration: 0.15 })
```

## Per-layer CSS filters

```typescript
// Apply a filter to a named render layer:
this._post.setLayerFilter('Background', { type: 'blur', radius: 8 })
this._post.setLayerFilter('UI', { type: 'colour-grade', saturation: 0.0 }) // greyscale

// Toggle without removing:
this._post.toggleLayerFilter('Background', false) // disable
this._post.toggleLayerFilter('Background', true)  // re-enable

// Read:
const f: LayerFilterOptions | undefined = this._post.getLayerFilter('Background')

// Remove:
this._post.clearLayerFilter('Background')
```

## Effect types

| PostEffectType | Key params | Description |
|---|---|---|
| `'bloom'` | `threshold`, `strength`, `radius` | Bright-pass additive glow |
| `'vignette'` | `intensity` | Screen-edge darkening |
| `'chromatic-aberration'` | `offset` | RGB channel split |
| `'blur'` | `strength` | Gaussian blur |
| `'pixelate'` | `size` | Nearest-neighbour downscale |
| `'scanlines'` | `spacing` | CRT scanline overlay |
| `'colour-grade'` | `lut`, `saturation`, `brightness`, `contrast` | Global tone control |
| `'outline'` | `thickness`, `colour` | Object outline |
| `'shockwave'` | `x`, `y`, `radius`, `amplitude` | Ripple distortion |
| `'noise'` | `intensity` | Film grain |

## Layer filter types

| LayerFilterType | Notes |
|---|---|
| `'blur'` | Params: `radius` (px) |
| `'brightness'` | Params: `value` (0–2, 1 = identity) |
| `'contrast'` | Params: `value` (0–2) |
| `'saturate'` | Params: `value` (0–2) |
| `'hue-rotate'` | Params: `degrees` |
| `'colour-grade'` | Params: `saturation`, `contrast`, `value` |
| `'outline'` | Params: `colour`, `thickness` |
| `'invert'` | No params |

## API reference

| Method | Signature | Notes |
|---|---|---|
| `add` | `(type: PostEffectType, opts?: PostEffectOptions): this` | Adds or replaces the effect. Chainable. |
| `remove` | `(type: PostEffectType): void` | Remove by type. |
| `has` | `(type: PostEffectType): boolean` | Whether the effect is active. |
| `get` | `(type: PostEffectType): ActiveEffect \| undefined` | Read current options. |
| `flash` | `(opts?: { colour?: number, duration?: number }): void` | Transient bright flash. |
| `setLayerFilter` | `(layerId: string, filter: LayerFilterOptions): void` | Apply CSS filter to a layer. |
| `clearLayerFilter` | `(layerId: string): void` | Remove layer filter. |
| `toggleLayerFilter` | `(layerId: string, enabled: boolean): void` | Enable/disable without removing. |
| `getLayerFilter` | `(layerId: string): LayerFilterOptions \| undefined` | Read current filter. |
| `update` | `(dt: number): void` | Decay transient effects. Call every frame. |
| `clear` | `(): void` | Remove all effects and reset transitions. |
| `destroy` | `(): void` | Clear and release. Call in onDestroy. |

## Notes

- Effects composite in add order — `bloom` before `colour-grade` grades the bloomed result.
- `PostProcessSystem` uses a second WebGL framebuffer. On `'potato'` and `'low'` GPU tiers, disable `bloom` and `blur`.
- Per-layer filters use CSS compositing; they do not require the WebGL framebuffer.
