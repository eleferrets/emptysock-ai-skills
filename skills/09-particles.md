# Particles

`ParticleEmitter` is a component that spawns and animates short-lived visual particles. Attach it to an entity and configure it before calling `start()`.

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

The texture is sampled once per particle at spawn time. Animated sprite-sheets are not supported — use `textureName` with a single-frame sprite for best performance.

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

`ParticleEmitter` holds an internal particle pool. Call `stop()` in `onDestroy` and then destroy the entity to release the pool:

```typescript
override onDestroy(): void {
  const emitter = this.entity.getComponent(ParticleEmitter)
  emitter?.stop()
  // entity.destroy() is called automatically by the scene
}
```

Forgetting to stop an emitter before scene teardown does not leak memory — the component is destroyed with its entity — but stopping it first prevents a frame of stray particle updates.

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

- Keep `count` as low as visually acceptable — each particle is updated every frame.
- Prefer `burst()` for one-shot effects rather than `start()` + `stop()` with a very short duration.
- Particles share a single draw call when they share the same `textureName` (or both use the default). Mixing textures across emitters adds extra draw calls.
- Destroy emitter entities when they are no longer needed; do not leave stopped emitters alive in the scene indefinitely.
