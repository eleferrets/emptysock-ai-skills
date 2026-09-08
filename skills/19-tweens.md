# Tween

`Tween` animates object properties over time with easing curves. All methods are static — no instantiation needed.

## Import

```typescript
import { Tween, type TweenOptions } from '@emptysock/engine'
```

## Basic tween

```typescript
import { Tween } from '@emptysock/engine'

// Tween any plain object's numeric properties:
const handle = Tween.to(entity.position, { x: 400, y: 200 }, {
  duration: 1.0,
  ease:     'sineInOut',
})
```

## `Tween.from` — animate from a value to current

```typescript
// Start the sprite at alpha 0 and fade in to its current alpha:
Tween.from(sprite, { alpha: 0 }, { duration: 0.5, ease: 'sineOut' })
```

## Chaining with `Tween.sequence`

```typescript
// Run tweens one after another:
const seq = Tween.sequence([
  Tween.to(entity.position, { x: 200 }, { duration: 0.4, ease: 'quadOut' }),
  Tween.to(entity.position, { y: 100 }, { duration: 0.3, ease: 'quadIn'  }),
])
```

## Options

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `duration` | `number` | required | Seconds |
| `ease` | `string` | `'linear'` | See easings list below |
| `delay` | `number` | `0` | Seconds before tween starts |
| `loop` | `boolean` | `false` | Repeat indefinitely |
| `yoyo` | `boolean` | `false` | Reverse on each alternate iteration (use with `loop`) |
| `onComplete` | `() => void` | — | Called once when the tween finishes |

## Cancellation

```typescript
const handle = Tween.to(entity.position, { x: 500 }, { duration: 2.0 })
// Cancel before it finishes:
handle.cancel()
```

Always cancel running tweens in `onDestroy()` if they reference scene objects:

```typescript
private _moveTween: TweenHandle | null = null

override onLoad(): void {
  this._moveTween = Tween.to(this.entity.position, { x: 600 }, { duration: 3.0, loop: true })
}

override onDestroy(): void {
  this._moveTween?.cancel()
}
```

## Easings

| Name | Curve |
|------|-------|
| `linear` | Constant speed |
| `sineIn` / `sineOut` / `sineInOut` | Gentle S-curve |
| `quadIn` / `quadOut` / `quadInOut` | Moderate acceleration |
| `cubicIn` / `cubicOut` / `cubicInOut` | Stronger acceleration |
| `bounceOut` / `bounceIn` | Springy bounce at end/start |
| `elasticOut` | Overshoot and spring back |
| `backOut` / `backIn` | Slight overshoot |

## API reference

| Method | Signature | Description |
|--------|-----------|-------------|
| `Tween.to` | `(target: object, props: object, opts: TweenOptions): TweenHandle` | Animate `target` properties to the given values. |
| `Tween.from` | `(target: object, props: object, opts: TweenOptions): TweenHandle` | Animate from the given values to the current values. |
| `Tween.sequence` | `(tweens: TweenHandle[]): TweenHandle` | Run tweens in order; returns a handle for the whole sequence. |

## Notes

- `Tween` only animates **numeric** properties. String or boolean properties are ignored.
- Do not call `Tween.to` inside `onUpdate()` every frame — call it once and store the handle.
- `Tween.sequence` starts each tween as the previous one finishes; all tweens in the array must be created before passing to `sequence`.
- For frame-rate-independent delays, prefer `Timer.after()` or coroutines (`yield waitSeconds(n)`) over the tween `delay` option.
