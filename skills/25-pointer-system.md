# PointerSystem

**Use this when** you want one input stream that covers mouse, touch, and pen without three separate code paths, plus gestures (tap, long-press, swipe, pinch) and wheel/trackpad classification. `PointerSystem` unifies all of that via native Pointer Events, and `InputManager` surfaces the result as `input.pointers`, `input.gestures`, and `input.wheelEvents` (see `skills/05-touch-input.md`).

---

## Reading pointers, gestures and wheel from InputManager

The game's `InputManager` (`game.input` / `ctx.input`) owns a `PointerSystem` and exposes its per-frame frozen results. `PointerSystem` itself is also exported from `@emptysock/engine` (with `MIN_TOUCH_TARGET_SIZE`), but you rarely construct one; the data types are (`PointerState`, `Gesture`, `TapGesture`, `LongPressGesture`, `SwipeGesture`, `PinchGesture`, `WheelEventInfo`).

```typescript
import type { PointerState, Gesture, WheelEventInfo } from '@emptysock/engine'

// Bootstrap (host code, once): input.attach() starts the pointer listeners alongside the keyboard ones.

// In onUpdate:
const pointers: ReadonlyArray<PointerState> = input.pointers
const primary = pointers.find((p) => p.isPrimary)   // mouse, or first active touch
if (primary !== undefined) moveCrosshair(primary.x, primary.y)

for (const p of pointers) {
  // p.pointerType: 'mouse' | 'touch' | 'pen' | 'unknown'
}
```

`PointerState`: `id`, `x`, `y`, `dx`, `dy`, `startX`, `startY`, `startTime`, `pointerType`, `isPrimary`, `buttons`.

## Gestures

`input.gestures` is the list of gestures recognised since the previous frame's snapshot (`ReadonlyArray<Gesture>`):

```typescript
for (const g of input.gestures) {
  switch (g.type) {
    case 'tap':
      selectAt(g.x, g.y)
      break
    case 'longpress':
      openContextMenu(g.x, g.y)
      break
    case 'swipe':
      if (g.direction === 'left') nextPage()   // also g.velocity, g.distance
      break
    case 'pinch':
      zoom *= g.deltaScale                     // also g.scale, g.distance
      break
  }
}
```

Long-press detection is polled against the game loop rather than a `setTimeout`, so it stays accurate even when the frame rate drops.

## Wheel / trackpad

```typescript
for (const w of input.wheelEvents) {
  if (w.isPinchZoom) {
    zoom *= 1 - w.deltaY * 0.01
  } else if (w.source === 'trackpad') {
    pan(w.deltaX, w.deltaY)
  } else {
    zoom *= w.deltaY > 0 ? 0.9 : 1.1   // discrete mouse-wheel click ('mouse-wheel')
  }
}
```

`source` is `'trackpad' | 'mouse-wheel'`, a best guess: trackpads deliver small fractional deltas continuously, mouse wheels deliver large discrete deltas per click.

---

## Touch targets

Keep touch-interactive widgets at least 44 CSS pixels square. There is no exported constant for this; use the literal 44 and feed `ViewportSystem`'s computed scale into your layout (see `skills/24-viewport-system.md`).

---

## Rules

| Wrong | Right |
|---|---|
| Listening for `touchstart`/`mousedown` separately | Read `input.pointers`: one stream covers mouse, touch, and pen |
| Using `setTimeout` to detect a long-press | Read `input.gestures` for `'longpress'` |
| Assuming `ctrlKey` on a wheel event means Ctrl is actually held | Browsers fake `ctrlKey: true` for trackpad pinch-to-zoom; check `isPinchZoom` |
| Sizing touch controls below 44px | Use 44px as a floor |
| Forgetting `input.detach()` when tearing the game down | Call it; it removes the native listeners |
