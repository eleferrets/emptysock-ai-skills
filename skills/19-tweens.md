# TweenManager

`TweenManager` animates numeric object properties over time with easing curves, and provides scene-local timers. Create one instance per scene, call `update(dt)` every frame, and it is garbage-collected with the scene — no explicit teardown needed.

## Import

```typescript
import { TweenManager, type TweenOptions, type EasingName } from '@emptysock/engine'
```

## Setup (in onLoad)

```typescript
private _tweens!: TweenManager

override onLoad(): void {
  this._tweens = new TweenManager()
}

override onUpdate(dt: number): void {
  this._tweens.update(dt)   // must be called every frame
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

## API reference

| Method | Signature | Description |
|--------|-----------|-------------|
| `to` | `(target: Record<string, number>, props: Record<string, number>, opts: TweenOptions): void` | Animate target properties from current values to props. |
| `after` | `(seconds: number, fn: () => void): void` | Run fn once after seconds. |
| `every` | `(seconds: number, fn: () => void): void` | Run fn repeatedly every seconds until scene unloads. |
| `update` | `(dt: number): void` | Advance all tweens and timers. Call once per frame in onUpdate. |

## Notes

- `TweenManager` only animates **numeric** properties. Non-numeric properties are silently ignored.
- Do not call `to()` inside `onUpdate()` every frame — call it once when you want to start a tween.
- For complex sequences, use coroutines (`yield waitSeconds(n)`) — they express multi-step time logic more clearly than chained `onComplete` callbacks.
- `after()` and `every()` timers are tied to this `TweenManager` instance; they stop automatically when the scene is done.
