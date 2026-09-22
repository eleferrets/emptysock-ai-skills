# TweenManager

**Use this when** you need something to smoothly animate over time, or a timer that cleans itself up with the scene. `TweenManager` animates numeric object properties with easing curves and doubles as a scene-local timer. Make one per scene, call `update(dt)` every frame, and forget about teardown — it's garbage-collected along with the scene.

## Import

```typescript
import { TweenManager, type TweenOptions, type EasingName } from '@emptysock/engine'
```

## Setup (in onLoad)

```typescript
private _tweens: TweenManager | null = null

override onLoad(): void {
  this._tweens = new TweenManager()
}

override onUpdate(dt: number): void {
  this._tweens?.update(dt)   // must be called every frame
}
```

## Animate to a target value

```typescript
// Tween any plain object's numeric properties to new values:
this._tweens.to(entity.position, { x: 400, y: 200 }, {
  duration: 1.0,
  ease:     'sineInOut',
})
```

## Tween with delay and completion callback

```typescript
this._tweens.to(entity.position, { x: 600 }, {
  duration:   0.8,
  ease:       'quadOut',
  delay:      0.3,
  onComplete: () => { entity.addTag('arrived') },
})
```

## Scene-local timers

`TweenManager` doubles as a scene-local timer — no handles to cancel, no memory leaks when the scene unloads:

```typescript
// Run once after 2 seconds:
this._tweens.after(2.0, () => { this.spawnWave() })

// Run on a repeating interval:
this._tweens.every(5.0, () => { this.spawnPowerUp() })
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
const handle = this._tweens.to(entity.position, { x: 600 }, { duration: 1.0 })
// Later, if needed:
handle.cancel()

const timer = this._tweens.every(2.0, () => { this.spawnEnemy() })
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

## Notes

- `TweenManager` only touches **numeric** properties. Anything else is silently skipped.
- Don't call `to()` every frame inside `onUpdate()` — call it once, when you actually want the tween to start.
- For anything with multiple steps, coroutines (`yield waitSeconds(n)`) read a lot more clearly than a pile of chained `onComplete` callbacks.
- `after()` and `every()` timers belong to the `TweenManager` instance that created them — they stop automatically once the scene is done.
