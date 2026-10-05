# PostProcessSystem

**Use this when** you want screen-wide visual effects (bloom, vignette, a hit-flash, a scene transition overlay) or a filter on one render layer. `PostProcessSystem` is a framework-agnostic registry of effects; `RenderPipeline` reads it to apply real filters once you call `renderPipeline.attachPostProcess(post)`. Call `update(dt)` every frame so transient effects (flashes, shockwaves) decay.

## Availability

`PostProcessSystem` and its option types (`PostEffectType`, `PostEffectOptions`, `LayerFilterOptions`, `LayerFilterType`, `ColourblindMode` and the per-effect option interfaces) are exported from `@emptysock/engine`, together with `COLOURBLIND_MATRICES`, `colourblindFilterId` and `colourblindFilterDefsSVG`. It is what `RenderPipeline.attachPostProcess()` and `SceneTransitionManager.attachPostProcess()` accept.

## Setup

```typescript
import { PostProcessSystem } from '@emptysock/engine'

const post = new PostProcessSystem()
// once, where you build the pipeline:  render.attachPostProcess(post)

export const Level = defineScene({
  onUpdate(dt) {
    post.update(dt)   // drives transient effect (flash, shockwave) decay
  },
  onUnload() {
    post.clear()      // remove effects; call post.destroy() when discarding it
  },
})
```

## Full-screen effects

`add()` returns `this` for method chaining:

```typescript
// Add effects — composited in the order they were added:
post
  .add('bloom', { threshold: 0.7, strength: 0.4 })
  .add('vignette', { intensity: 0.45 })

// Check / remove:
if (post.has('bloom')) {
  post.remove('bloom')
}

// Read the current options for an active effect:
const bloomEffect = post.get('bloom') // ActiveEffect | undefined
```

## Transient flash

```typescript
post.flash({ colour: 0xffffff, duration: 0.15 })
```

## Per-layer CSS filters

```typescript
// Apply a filter to a named render layer:
post.setLayerFilter('Background', { type: 'blur', radius: 8 })
post.setLayerFilter('UI', { type: 'colour-grade', saturation: 0.0 }) // greyscale

// Toggle without removing:
post.toggleLayerFilter('Background', false) // disable
post.toggleLayerFilter('Background', true)  // re-enable

// Read:
const f: LayerFilterOptions | undefined = post.getLayerFilter('Background')

// Remove:
post.clearLayerFilter('Background')
```

## Effect types

| PostEffectType | Key params | Description |
|---|---|---|
| `'bloom'` | `threshold`, `strength` | Bright-pass additive glow |
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
| `'colourblind'` | Params: `mode` (`'protanopia'`, `'deuteranopia'`, `'tritanopia'`); a simulation, see `skills/28-accessibility-debugging.md` |
| `'rain-glass'` | Params: `intensity`, `dropletSize`, `dropletSpeed`, `streakAmount`, `quality`, `fog`, `blur`, `slope`, `wind`, `seed`, `wiperEnabled`, `wiperPeriod`; see `skills/39-rain-effects.md` |
| `'none'` | No filter |

## API reference

| Method | Signature | Notes |
|---|---|---|
| `add` | `(type: PostEffectType, opts?: PostEffectOptions): this` | Adds or replaces the effect. Chainable. |
| `remove` | `(type: PostEffectType): void` | Remove by type. |
| `has` | `(type: PostEffectType): boolean` | Whether the effect is active. |
| `get` | `(type: PostEffectType): ActiveEffect \| undefined` | Read current options. |
| `flash` | `(opts?: { colour?: number, duration?: number }): void` | Transient bright flash. |
| `setLayerFilter` | `(layerId: string, filter: LayerFilterOptions): void` | Apply a filter to a layer. |
| `clearLayerFilter` | `(layerId: string): void` | Remove layer filter. |
| `toggleLayerFilter` | `(layerId: string, enabled: boolean): void` | Enable/disable without removing. |
| `getLayerFilter` | `(layerId: string): LayerFilterOptions \| undefined` | Read current filter. |
| `update` | `(dt: number): void` | Decay transient effects. Call every frame. |
| `clear` | `(): void` | Remove all effects and reset transitions. |
| `destroy` | `(): void` | Clear and release. Call in onDestroy. |

## Notes

- Effects composite in the order you added them: `bloom` before `colour-grade` means the grade acts on the already-bloomed result.
- Other members: `effects` (all active effects), `layerFilters` (map of layer filters), `flashActive` / `flashIntensity` / `flashColour`, `transitionActive`, `beginTransition(effect, colour?)` / `endTransition()`, `cssFilterForLayer(layerId)`.
- `bloom` and `blur` are the expensive effects; turn them off on the `'potato'` and `'low'` GPU tiers (see `skills/24-viewport-system.md`).
- Layer filters are applied by `RenderPipeline` each frame once `attachPostProcess(post)` has been called.

## Scene transitions

`SceneTransitionManager` (construct your own; it is not a singleton) only times a transition and calls your `load` callback; it never imports pixi, so it stays inside the engine's environment boundary. Attach a `PostProcessSystem` as its sink and let `RenderPipeline` paint the overlay each frame:

```typescript
import { PostProcessSystem, SceneTransitionManager } from '@emptysock/engine'

const post = new PostProcessSystem()
const transitions = new SceneTransitionManager()
transitions.attachPostProcess(post)

// Start a transition: `load` fires once `duration` (default 0.3s) has elapsed
transitions.transition(() => game.loadScene(NextScene), { effect: 'fade', duration: 0.4, colour: 0x000000 })

// Every frame:
transitions.update(dt)
post.update(dt)
// RenderPipeline.renderFrame() paints the overlay automatically once render.attachPostProcess(post) was called;
// call render.renderTransitionOverlay(post) yourself only in a custom render loop
```

`transition(load, options?)` options: `effect` (`'none' | 'fade' | 'wipe' | 'slide'`, default `'none'`), `duration` (seconds), `colour`. A second `transition()` call while one is in flight replaces the pending `load`. `transitions.isTransitioning` reports state. The overlay is a full-screen rect (a triangle-wave alpha for `'fade'`, a growing rect for `'wipe'`, a sweeping rect for `'slide'`), not a true two-scene crossfade.
