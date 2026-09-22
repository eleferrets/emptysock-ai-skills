# SequenceSystem

**Use this when** you're playing back a keyframed animation authored in the IDE's **Sequence Editor**, or building keyframe playback in code. `SequenceSystem` plays a `SequenceDefinition` — a list of tracks, each a target property plus keyframes and an optional easing — against a plain numeric target object, by scheduling real `TweenManager.to()` calls under the hood. A sequence exported from the panel and a hand-written `TweenManager` chain play back identically, because `play()` schedules exactly the calls you'd write by hand anyway.

---

## Types

```typescript
interface SequenceKeyframe {
  time: number
  value: number
}

interface SequenceTrackDef {
  property: string          // key set on the target object, e.g. 'x', 'rotation'
  keyframes: SequenceKeyframe[]
  ease?: EasingName          // defaults to 'linear'
}

interface SequenceDefinition {
  duration: number
  tracks: SequenceTrackDef[]
}
```

---

## Usage

```typescript
import { TweenManager, SequenceSystem } from '@emptysock/engine'

// One TweenManager per scene — create in onLoad, destroy in onDestroy.
private _tweens = new TweenManager()
private _seq = new SequenceSystem()

override onLoad(): void {
  const target = { x: 0, opacity: 0 }
  const def: SequenceDefinition = {
    duration: 2,
    tracks: [
      { property: 'x', keyframes: [{ time: 0, value: 0 }, { time: 2, value: 200 }], ease: 'bounceOut' },
      { property: 'opacity', keyframes: [{ time: 0, value: 0 }, { time: 0.5, value: 1 }] },
    ],
  }
  this._seq.play(this._tweens, target, def)
}

override onUpdate(dt: number): void {
  this._tweens.update(dt)   // required — SequenceSystem schedules through this
}

override onDestroy(): void {
  this._seq.stop()          // cancels all scheduled tweens
  this._tweens.destroy()
}
```

Resume mid-sequence by passing `startAt`:

```typescript
this._seq.play(this._tweens, target, def, /* startAt */ 1.5)
```

`play(tweens, target, def, startAt)` sets `target[property]` to the exact value at `startAt` right away, then schedules one `to()` per remaining keyframe segment with `delay` measured from `startAt` — resuming mid-playback never flashes the wrong value first.

---

## Scrubbing without a TweenManager

`evaluateTrackAt(track, time)` is a pure, side-effect-free evaluation of a single track at an arbitrary time, using the same per-segment easing math `play()` schedules. Use it for a preview scrubber or a timeline UI that needs to sample a value without touching playback state:

```typescript
import { evaluateTrackAt } from '@emptysock/engine'

const previewX = evaluateTrackAt(def.tracks[0], scrubberTime)
```

---

## Rules

| Wrong | Right |
|---|---|
| Forgetting `tweens.update(dt)` in `onUpdate` | `SequenceSystem.play()` only schedules `TweenManager` calls — nothing moves without `update(dt)` |
| Calling `play()` again mid-playback to "restart" | Call `seq.stop()` first, or pass the current elapsed time as `startAt` to resume cleanly |
| Sampling a value for a scrubber via `play()` + `stop()` on every drag frame | Use `evaluateTrackAt()` instead — no side effects, no `TweenManager` involved |
| Assuming keyframes must be sorted or evenly spaced | Keyframes sort by `time`; the segments between them can be any length with any `ease` |
