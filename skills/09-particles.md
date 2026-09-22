# Particles

**Use this when** you need sparks, smoke, confetti, or any other short-lived visual flourish. `ParticleEmitter` is a component that spawns and animates those particles for you — attach it to an entity, configure it, then call `start()`.

---

## Creating and configuring an emitter

```typescript
import { ParticleEmitter } from '@emptysock/engine'

// In onLoad:
const burst = scene.createEntity('confetti-emitter')
const emitter = burst.addComponent(ParticleEmitter, {
  rate:        30,          // particles per second (continuous mode)
  lifetime:    2.5,         // seconds each particle lives
  speed:       150,         // initial speed in world units/second
  spread:      45,          // emission cone half-angle in degrees
  count:       200,         // max simultaneous particles
})
emitter.start()
```

All constructor options are also settable as properties before `start()` is called:

```typescript
emitter.rate     = 60
emitter.lifetime = 1.0
emitter.speed    = 200
emitter.spread   = 90
emitter.count    = 500
```

---

## Sprite-sheet particles

Point `textureName` at a registered texture asset to use a custom sprite instead of the default white square:

```typescript
const emitter = entity.addComponent(ParticleEmitter, {
  rate:        20,
  lifetime:    1.5,
  speed:       80,
  spread:      30,
  count:       100,
  textureName: 'fx/sparkle.png',   // path relative to the assets/ folder
})
emitter.start()
```

The texture is sampled once per particle at spawn time. Animated sprite-sheets aren't a thing here — stick to a single-frame sprite for `textureName` if you want good performance.

---

## Starting and stopping

```typescript
emitter.start()    // begin continuous emission at `rate` particles/s
emitter.stop()     // stop emitting new particles; existing ones finish their lifetime
emitter.burst(50)  // fire exactly 50 particles immediately, ignores rate
```

`burst(count)` does not require `start()` to have been called first. It is useful for one-shot effects like explosions or pickups.

---

## Cleaning up in onDestroy

`ParticleEmitter` keeps an internal particle pool alive. Call `stop()` in `onDestroy`, then let the entity get destroyed to release it:

```typescript
override onDestroy(): void {
  const emitter = this.entity.getComponent(ParticleEmitter)
  emitter?.stop()
  // entity.destroy() is called automatically by the scene
}
```

Forgetting to stop an emitter before scene teardown won't leak memory (the component dies with its entity), but stopping it first avoids one stray frame of particle updates on the way out.

---

## Example: basic confetti emitter

```typescript
import { Scene, ParticleEmitter, type SceneConfig } from '@emptysock/engine'

export class ConfettiScene extends Scene {
  static readonly config: SceneConfig = { renderMode: '2d', gameSpeed: 60 }

  private emitter: ParticleEmitter | undefined

  override async onLoad(): Promise<void> {
    const entity = this.createEntity('confetti')
    this.emitter = entity.addComponent(ParticleEmitter, {
      rate:     40,
      lifetime: 3.0,
      speed:    120,
      spread:   180,
      count:    300,
    })
    this.emitter.start()
  }

  override onUpdate(_dt: number): void {
    // nothing — emitter runs automatically
  }

  override onDestroy(): void {
    this.emitter?.stop()
  }
}
```

---

## Example: sprite explosion burst

```typescript
import { ParticleEmitter, Timer } from '@emptysock/engine'

// Called when the player collects a coin:
private spawnCoinExplosion(x: number, y: number): void {
  const entity = this.createEntity('coin-explosion')
  entity.position = { x, y }

  const emitter = entity.addComponent(ParticleEmitter, {
    lifetime:    0.8,
    speed:       220,
    spread:      360,
    count:       24,
    textureName: 'fx/coin-shard.png',
  })

  emitter.burst(24)

  // destroy the emitter entity after the longest particle lifetime
  Timer.after(1.0, () => entity.destroy())
}
```

---

## Performance notes

- Keep `count` as low as it can be while still looking good — every particle gets updated every frame, and that adds up fast.
- Prefer `burst()` over `start()` + `stop()` on a tiny timer for one-shot effects. It's what `burst()` is for.
- Particles sharing the same `textureName` (or both on the default) share one draw call. Mix textures across emitters and you're paying for extra draw calls.
- Destroy emitter entities once they're done. A stopped emitter just sitting around in the scene isn't doing anyone any favors.
