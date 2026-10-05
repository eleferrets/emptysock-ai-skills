# Particles

**Use this when** you need sparks, smoke, confetti, rain, or any other short-lived visual flourish. `ParticleEmitter` is a plain, renderer-agnostic simulation class (not an ECS component). Create one directly, step it every frame, and mount it on a render layer with `RenderPipeline.mountParticles()` to draw it.

---

## Creating and configuring an emitter

```typescript
import { ParticleEmitter } from '@emptysock/engine'

const emitter = new ParticleEmitter({
  texture: 'fx/confetti.png',                 // optional; default is a white square
  emissionRate: 30,                           // particles per second (continuous mode)
  lifetime: { min: 1.5, max: 2.5 },           // seconds each particle lives
  velocity: { x: { min: -100, max: 100 }, y: { min: -200, max: -80 } },
  acceleration: { x: 0, y: 150 },             // px/s^2 (gravity-like)
  maxParticles: 200,                          // max simultaneous particles
})
emitter.x = 400   // emitter position, writable
emitter.y = 300
```

Manage several emitters by keeping them in your own array and calling `update(dt)` on each, or let the exported `ParticleSystem` helper own them: `create(options?)` returns a `ParticleEmitter`, `remove(emitter)` drops one, `update(dt)` steps all, `clear()` / `destroy()` end them, and `emitters` lists the live ones.

### `ParticleEmitterOptions`

| Option | Default | Notes |
|--------|---------|-------|
| `texture` | `''` | Texture path; empty = white square |
| `emissionRate` | `20` | Particles per second while `active` |
| `lifetime` | `{ min: 0.5, max: 1.5 }` | Seconds |
| `velocity` | `x: -50..50, y: -100..-50` | Per-axis `{ min, max }` ranges |
| `acceleration` | `{ x: 0, y: 100 }` | px/s^2 |
| `startScale` / `endScale` | `1` / `0` | Scale ramp over lifetime |
| `sizeWiggle` | `0` | Per-step random scale jitter |
| `startAlpha` / `endAlpha` | `1` / `0` | Alpha ramp |
| `colorGradient` | `[0xffffff]` | Colours sampled over lifetime |
| `shape` | `'point'` | `'point' \| 'circle' \| 'rectangle' \| 'line'` |
| `shapeRadius` / `shapeWidth` / `shapeHeight` | `0` | Spawn area for the shape |
| `rotationSpeed` | `0` | Radians per second |
| `maxParticles` | `500` | Pool cap |
| `speedWiggle` / `dirWiggle` | `0` | Per-step random speed / direction (degrees) jitter |
| `blendMode` | `'normal'` | `'normal' \| 'add'` |

---

## Driving and drawing

```typescript
// Every frame, step the simulation:
emitter.update(dt)

// Draw it: mount onto a layer of your RenderPipeline (async; loads the texture)
await render.mountParticles(emitter, 'default')
// ...and when finished:
render.unmountParticles(emitter)
```

---

## Starting and stopping

```typescript
emitter.active = true   // default: emits continuously at emissionRate while you call update()
emitter.stop()          // sets active = false: no new particles; existing ones finish their lifetime
emitter.emit(50)        // burst: spawn exactly 50 particles immediately, ignores emissionRate
emitter.clear()         // drop every live particle
emitter.activeCount     // live particle count
```

`emit(count)` works whether or not the emitter is `active`, so a one-shot effect is `new ParticleEmitter({...})`, `stop()`, then `emit(n)`.

---

## Example: sprite explosion burst

```typescript
function spawnCoinExplosion(x: number, y: number): void {
  const burst = new ParticleEmitter({
    texture: 'fx/coin-shard.png',
    lifetime: { min: 0.5, max: 0.8 },
    velocity: { x: { min: -220, max: 220 }, y: { min: -220, max: 220 } },
    maxParticles: 24,
  })
  burst.x = x
  burst.y = y
  burst.stop()      // no continuous emission
  burst.emit(24)
  void render.mountParticles(burst, 'default')   // and call burst.update(dt) each frame until it is unmounted
  // Remove it after the longest lifetime (use TweenManager.after):
  tweens.after(1.0, () => { render.unmountParticles(burst) })
}
```

---

## Rain preset

`rainParticlePreset({ width?, density?, wind?, texture? })` returns ready-made `ParticleEmitterOptions` (a line emitter across the top of the view); see `skills/39-rain-effects.md`.

---

## Cleaning up

Call `render.unmountParticles(emitter)` on scene unload and drop your references. Particles are plain data, but a mounted pixi container stays on the stage until unmounted.

## Performance notes

- Keep `maxParticles` as low as it can be while still looking good; every live particle updates each frame.
- Prefer `emit(n)` with `stop()` for one-shot effects over a short-lived continuous emitter.
- Emitters sharing one texture are cheaper than many different textures.
- Remove emitters once they're done.
