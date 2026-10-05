# TweenManager

**Use this when** you need something to smoothly animate over time, or a timer tied to a scene. `TweenManager` animates numeric object properties with easing curves and doubles as a timer. Make one per scene, call `update(dt)` every frame, and call `killAll()` / `destroy()` on unload.

## Import

```typescript
import { TweenManager, Transform, type TweenOptions, type EasingName } from '@emptysock/engine'
```

## Setup

```typescript
import { defineScene, TweenManager } from '@emptysock/engine'

let tweens: TweenManager | null = null

export const Level = defineScene({
  onLoad() { tweens = new TweenManager() },
  onUpdate(dt) { tweens?.update(dt) },   // must be called every frame
  onUnload() { tweens?.destroy(); tweens = null },
})
```

## Animate to a target value

```typescript
// Tween any object's numeric properties (an entity's Transform proxy works):
const t = entity.get(Transform)
if (t !== undefined) tweens.to(t, { x: 400, y: 200 }, {
  duration: 1.0,
  ease:     'sineInOut',
})
```

## Tween with delay and completion callback

```typescript
tweens.to(t, { x: 600 }, {
  duration:   0.8,
  ease:       'quadOut',
  delay:      0.3,
  onComplete: () => { arrived = true },
})
```

## Scene-local timers

`TweenManager` doubles as a timer. Timers only run while you call `update(dt)`, and `killAll()` / `destroy()` clears them when the scene unloads:

```typescript
// Run once after 2 seconds:
tweens.after(2.0, () => { spawnWave() })

// Run on a repeating interval:
tweens.every(5.0, () => { spawnPowerUp() })
```

## Options

| Option       | Type           | Default    | Notes                            |
|-------------|----------------|------------|----------------------------------|
| `duration`  | `number`       | required   | Seconds                          |
| `ease`      | `EasingName`   | `'linear'` | See easings list below           |
| `delay`     | `number`       | `0`        | Seconds before tween starts      |
| `onComplete`| `() => void`   | —          | Called once when tween finishes  |

## Easings

| Name | Curve |
|------|-------|
| `linear` | Constant speed |
| `sineIn` / `sineOut` / `sineInOut` | Gentle S-curve |
| `quadIn` / `quadOut` / `quadInOut` | Moderate acceleration |
| `cubicIn` / `cubicOut` / `cubicInOut` | Stronger acceleration |
| `bounceOut` | Springy bounce at the end |
| `elasticOut` | Overshoot and spring back |

## Cancelling a tween or timer

All three methods return a `TweenHandle` with a `cancel()` method:

```typescript
const handle = tweens.to(t, { x: 600 }, { duration: 1.0 })
// Later, if needed:
handle.cancel()

const timer = tweens.every(2.0, () => { spawnEnemy() })
// Stop spawning:
timer.cancel()
```

## API reference

| Method | Signature | Description |
|--------|-----------|-------------|
| `to` | `(target: Record<string, number>, props: Record<string, number>, opts: TweenOptions): TweenHandle` | Animate target properties. Returns a handle to cancel. |
| `after` | `(seconds: number, fn: () => void): TweenHandle` | Run fn once after seconds. Returns a handle to cancel. |
| `every` | `(seconds: number, fn: () => void): TweenHandle` | Run fn on a repeating interval. Returns a handle to cancel. |
| `update` | `(dt: number): void` | Advance all tweens and timers. Call once per frame in onUpdate. |
| `killAll` | `(): void` | Cancel every pending tween and timer. |
| `destroy` | `(): void` | Tear down the manager. |

## Notes

- `TweenManager` only touches **numeric** properties. Anything else is silently skipped.
- Don't call `to()` every frame inside `onUpdate()` — call it once, when you actually want the tween to start.
- For anything with multiple steps, coroutines (`yield waitSeconds(n)`) read a lot more clearly than a pile of chained `onComplete` callbacks.
- `after()` and `every()` timers belong to the `TweenManager` instance that created them; they only advance while you call `update(dt)`, so stop updating (or call `killAll()`) when the scene is done.
