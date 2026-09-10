# PostProcessSystem

`PostProcessSystem` manages full-screen post-processing effects and per-layer filters. It is a static class — no instantiation required.

## Import

```typescript
import { PostProcessSystem, type PostEffectType, type LayerFilterType } from '@emptysock/engine'
```

## Full-screen effects

Add and remove full-screen effects by type:

```typescript
// Add a full-screen bloom effect (options are effect-specific):
PostProcessSystem.add('bloom', { threshold: 0.7, intensity: 1.2 })

// Check whether an effect is active:
if (PostProcessSystem.has('bloom')) {
  PostProcessSystem.remove('bloom')
}
```

### Supported PostEffectType values

| Value | Description |
|-------|-------------|
| `'bloom'` | HDR bloom glow |
| `'vignette'` | Screen-edge darkening |
| `'chromatic-aberration'` | RGB channel split |
| `'scanlines'` | CRT scanline overlay |
| `'pixelate'` | Low-resolution pixelation |
| `'blur'` | Gaussian blur |
| `'grayscale'` | Desaturate the scene |

## Per-layer filters

Apply a filter to a single render layer without affecting others:

```typescript
// Apply a grayscale filter to the 'Background' layer:
PostProcessSystem.setLayerFilter('Background', 'grayscale')

// Toggle the filter on or off:
PostProcessSystem.toggleLayerFilter('Background', false) // disable
PostProcessSystem.toggleLayerFilter('Background', true)  // re-enable

// Read the current filter on a layer:
const f: LayerFilterType | null = PostProcessSystem.getLayerFilter('Background')

// Remove the filter entirely:
PostProcessSystem.clearLayerFilter('Background')
```

### Supported LayerFilterType values

| Value | Description |
|-------|-------------|
| `'grayscale'` | Desaturate the layer |
| `'sepia'` | Warm sepia tone |
| `'invert'` | Colour inversion |
| `'blur'` | Gaussian blur on the layer |

## API reference

| Method | Signature | Notes |
|--------|-----------|-------|
| `PostProcessSystem.add` | `(type: PostEffectType, opts?: object): void` | Add a full-screen effect. No-op if already active. |
| `PostProcessSystem.remove` | `(type: PostEffectType): void` | Remove a full-screen effect. |
| `PostProcessSystem.has` | `(type: PostEffectType): boolean` | Whether the effect is currently active. |
| `PostProcessSystem.setLayerFilter` | `(layerId: string, filter: LayerFilterType): void` | Set a filter on one layer. |
| `PostProcessSystem.clearLayerFilter` | `(layerId: string): void` | Remove the filter from a layer. |
| `PostProcessSystem.toggleLayerFilter` | `(layerId: string, enabled: boolean): void` | Enable or disable a layer's filter without removing it. |
| `PostProcessSystem.getLayerFilter` | `(layerId: string): LayerFilterType \| null` | Read the current layer filter. |

## Notes

- Effects are applied in the order they were added. Order can matter for visual results (e.g. bloom before blur vs. after).
- Per-layer filters require the layer to be registered with `LayerSystem` before `setLayerFilter` is called.
- Disable expensive effects (`bloom`, `blur`) on `'potato'` and `'low'` GPU tiers — check `Engine.gpuTier` before adding them.
- `PostProcessSystem` has no instance state. It applies globally to the current scene's render pipeline.
