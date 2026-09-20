# PointerSystem

`PointerSystem` unifies mouse, touch, and pen input into a single pointer stream via native Pointer Events, plus a small built-in gesture recognizer (tap / long-press / swipe / pinch) and wheel/trackpad classification. It complements `InputSystem` (keyboard + mouse buttons) and `GamepadSystem` (controllers) — use whichever fits the interaction, not all three for the same input.

---

## Setup (in onLoad)

```typescript
import { PointerSystem } from '@emptysock/engine'

class GameScene extends Scene {
  private _pointer = new PointerSystem()

  override onLoad(): void {
    this._pointer.attach()   // defaults to window; pass a target element to scope it
  }

  override onUpdate(): void {
    this._pointer.update()   // polls for long-press — call once per frame
  }

  override onDestroy(): void {
    this._pointer.destroy()
  }
}
```

---

## Reading pointer state

```typescript
const primary = this._pointer.primaryPointer   // mouse, or first active touch
if (primary) {
  moveCrosshair(primary.x, primary.y)
}

for (const p of this._pointer.pointers) {
  // p.pointerType: 'mouse' | 'touch' | 'pen' | 'unknown'
}
```

## Gestures

```typescript
const unsubscribe = this._pointer.onGesture((g) => {
  switch (g.type) {
    case 'tap':
      selectAt(g.x, g.y)
      break
    case 'longpress':
      openContextMenu(g.x, g.y)
      break
    case 'swipe':
      if (g.direction === 'left') nextPage()
      break
    case 'pinch':
      camera.zoom *= g.deltaScale
      break
  }
})
```

`longpress` only fires from `update()` being called each frame — it is polled against the game loop's own timing rather than a `setTimeout`, so it stays correct even if the frame rate drops.

## Wheel / trackpad

```typescript
this._pointer.onWheel((w) => {
  if (w.isPinchZoom) {
    camera.zoom *= 1 - w.deltaY * 0.01
  } else if (w.source === 'trackpad') {
    camera.pan(w.deltaX, w.deltaY)
  } else {
    camera.zoom *= w.deltaY > 0 ? 0.9 : 1.1   // discrete mouse-wheel click
  }
})
```

`source` is a heuristic: trackpads deliver small fractional deltas continuously; mouse wheels deliver large, discrete deltas per click.

---

## Touch targets

`MIN_TOUCH_TARGET_SIZE` (44, in CSS pixels) is the minimum recommended interactive-element size for touch. Pair it with `UISystem.setScale()`, fed from `ViewportSystem`'s computed scale, so buttons stay tappable at any design resolution:

```typescript
import { MIN_TOUCH_TARGET_SIZE } from '@emptysock/engine'

const button = new ButtonWidget({
  width: Math.max(120, MIN_TOUCH_TARGET_SIZE),
  height: Math.max(40, MIN_TOUCH_TARGET_SIZE),
  label: 'Play',
})
```

---

## Rules

| Wrong | Right |
|---|---|
| Listening for `touchstart`/`mousedown` separately | Use `PointerSystem` — one event stream for mouse, touch, and pen |
| Using `setTimeout` to detect a long-press | Call `pointer.update()` every frame; long-press is polled against the game loop |
| Assuming `ctrlKey` on a wheel event means the Ctrl key is held | Browsers synthesize `ctrlKey: true` for trackpad pinch-to-zoom — read `isPinchZoom` instead |
| Sizing touch controls below 44px | Use `MIN_TOUCH_TARGET_SIZE` as a floor |
| Forgetting `pointer.destroy()` in `onDestroy` | Always call it — removes native listeners and clears tracked pointers |
