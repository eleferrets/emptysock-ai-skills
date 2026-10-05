# Rain effects: RainGlassFilter and rainParticlePreset

**Use this when** a scene is set in the rain. Two independent pieces, meant to be layered:

- **Rain on the glass** (`'rain-glass'` layer filter / `RainGlassFilter`): droplets and streaks on the "lens", drawn as a screen-space post-process. Each drop refracts the scene behind it (an inverted, magnified miniature), with a dark rim and a bright glint; sliding drops leave a thin wet trail.
- **Falling rain** (`rainParticlePreset`): ready-made `ParticleEmitter` options for streaks falling through the world.

---

## Rain on glass

One route is a layer filter on `PostProcessSystem`, attached to a `RenderPipeline`; the other is building the filter yourself with `createRainGlassFilter` (below). The filter-option shape is the same:

```typescript
import { PostProcessSystem } from '@emptysock/engine'

const post = new PostProcessSystem()
pipeline.attachPostProcess(post)

post.setLayerFilter('default', {
  type: 'rain-glass',
  intensity: 0.7,       // 0..1 droplet opacity and refraction strength (default 0.6)
  dropletSize: 0.12,    // cell size of the big-drop grid; smaller = more, smaller drops (default 0.12)
  dropletSpeed: 0.35,   // fall speed of sliding drops per second (default 0.35)
  streakAmount: 0.5,    // 0 = mostly static drops, 1 = mostly sliding drops with trails (default 0.5)
})
```

Update the options at any time by calling `setLayerFilter` again for the same layer (the effect is not rebuilt; its values are updated). Remove it with `post.clearLayerFilter('default')`. Time advances by itself, so drops slide and creep with no per-frame call from you.

To build the filter yourself, `createRainGlassFilter(options?)` returns a `RainGlassFilter` with `setOptions(partial)`, `setResolution(width, height)` (call when the target size changes) and `tick(dtSeconds)` (advance time once per frame), and you attach it by assigning it to a PixiJS container's `filters` array (for example `pipeline.stage.filters = [filter]`).

**What it is not.** It is a single fragment-shader pass: no per-drop simulation, no drops merging or running together, and no real lens geometry. It reads as "the screen is wet", not as a physical simulation. The cost scales with resolution only, never with the number of drops, so it is fine on mid-range GPUs. Profile it on your weakest target device before shipping alongside other heavy effects.

Drops read best against a busy, bright scene (lights, windows, contrast). On a flat dark background they are subtle. Raise `intensity` rather than `dropletSize` for a stronger look.

---

## Falling rain

```typescript
import { ParticleEmitter, rainParticlePreset } from '@emptysock/engine'

const rain = new ParticleEmitter(rainParticlePreset({
  width: 1280,      // width of the emission strip; usually the viewport width
  density: 300,     // drops per second
  wind: 0,          // px/s^2 sideways push; positive = right
  texture: 'rain-streak.png',   // optional; empty falls back to a plain white streak
}))
rain.x = 640
rain.y = -20        // just above the top edge of the view

await pipeline.mountParticles(rain, 'foreground')   // loads the texture and draws it; unmount with pipeline.unmountParticles(rain)

// each frame
rain.update(dt)
```

The preset is a line emitter across the top of the screen firing fast, slightly tilted, short-lived, low-alpha drops (capped at 1200 live particles). Every field of the returned `ParticleEmitterOptions` can be overridden by spreading it: `new ParticleEmitter({ ...rainParticlePreset(), startAlpha: 0.8 })`.

## Rules

| Wrong | Right |
|---|---|
| Faking rain with hundreds of individual sprites | `rainParticlePreset` on one `ParticleEmitter` |
| Calling `createRainGlassFilter` and never calling `setResolution` | Use the `'rain-glass'` layer filter, which handles it, or call `setResolution` on resize |
| Expecting drops to merge or run into each other | It is a screen-space approximation; keep expectations there |
| Stacking rain-glass with bloom and blur on a low GPU tier | Drop the other heavy effects (`skills/21-post-process.md`) |
