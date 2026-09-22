# PostProcessSystem

**Use this when** you want screen-wide visual effects — bloom, vignette, a hit-flash, a scene transition overlay — or a CSS filter on one render layer. `PostProcessSystem` handles both. One instance per scene, and call `update(dt)` every frame so transient effects (flashes, shockwaves) decay properly.

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

- Effects composite in the order you added them — `bloom` before `colour-grade` means the grade acts on the already-bloomed result.
- `PostProcessSystem` needs a second WebGL framebuffer to do its thing. On `'potato'` and `'low'` GPU tiers, turn off `bloom` and `blur` — they're the expensive ones.
- Per-layer filters ride on CSS compositing instead, so they skip the WebGL framebuffer entirely.

## Scene transitions

`SceneManagerInstance.transition()` (see `skills/00-quickstart.md`) only times and tracks a `TransitionEffect` (`'none' | 'fade' | 'wipe' | 'slide'`) — it never imports pixi, so it stays inside the engine's environment boundary. Attach a `PostProcessSystem` and let `RenderPipeline` paint the overlay each frame:

```typescript
import { SceneManagerInstance, PostProcessSystem, RenderPipeline } from '@emptysock/engine'

// In onLoad:
private _postProcess = new PostProcessSystem()
SceneManagerInstance.attachPostProcess(this._postProcess)

// In onUpdate:
this._postProcess.update(dt)
this._render.renderFrame(this, this._postProcess)   // paints the transition overlay too

// Or paint it yourself without going through renderFrame's second argument:
this._render.renderTransitionOverlay(this._postProcess)
```

`PostProcessSystem.transitionEffect` / `.transitionProgress` / `.transitionColour` are set by `transition()` through `attachPostProcess()` and read by `RenderPipeline.renderTransitionOverlay()`. This paints a full-screen overlay rect on top of the current frame (a triangle-wave alpha for `'fade'`, a growing rect for `'wipe'`, a sweeping rect for `'slide'`) — it is not a true two-scene crossfade, since `RenderPipeline` does not keep two scenes' sprites live at once.
