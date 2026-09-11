# Touch & Pointer Input

Touch events, pointer (mouse/first-touch) state, and keyboard input are all handled by `InputSystem`. Create one instance in `onLoad`, call `attach()`, and call `flush()` at the start of each `onUpdate` before reading state.

## Import and setup

```typescript
import { InputSystem, type TouchPoint } from '@emptysock/engine'

class GameScene extends Scene {
  private _input: InputSystem | null = null

  override onLoad(): void {
    const input = new InputSystem()
    input.attach()   // register listeners on window
    this._input = input
  }

  override onUpdate(dt: number): void {
    const input = this._input
    if (input === null) return
    input.flush()   // must be called before reading any state

    // ... read input state here
  }

  override onDestroy(): void {
    this._input?.detach()
    this._input = null
  }
}
```

## Reading touch state

```typescript
override onUpdate(dt: number): void {
  const input = this._input
  if (input === null) return
  input.flush()

  // All active touches:
  const touches: ReadonlyArray<TouchPoint> = input.touches

  if (touches.length > 0) {
    const primary = input.primaryTouch   // lowest-id touch, or undefined
    if (primary !== undefined) {
      console.log(primary.x, primary.y)  // canvas coordinates
    }
  }

  // Mouse pointer (also covers single-touch primary position):
  const x = input.mouseX
  const y = input.mouseY
  const held    = input.isMouseDown(0)     // left button held
  const clicked = input.isMousePressed(0)  // fired once on click/tap
}
```

## Virtual buttons via pointer position

Map screen regions to game actions using mouse coordinates:

```typescript
override onUpdate(dt: number): void {
  const input = this._input
  if (input === null) return
  input.flush()

  const HALF = GAME_WIDTH / 2
  const movingLeft  = input.isMouseDown(0) && input.mouseX < HALF
  const movingRight = input.isMouseDown(0) && input.mouseX >= HALF

  if (movingLeft)  { /* move left */ }
  if (movingRight) { /* move right */ }
}
```

## Multi-touch example

```typescript
override onUpdate(dt: number): void {
  const input = this._input
  if (input === null) return
  input.flush()

  for (const touch of input.touches) {
    // Each TouchPoint has: id, x, y, dx (delta from last frame), dy
    console.log(touch.id, touch.x, touch.y)
  }

  // Detect a new touch this frame:
  if (input.isTouchStarted()) { /* finger just landed */ }
  if (input.isTouchEnded())   { /* finger just lifted */ }
}
```

## API reference

| Member | Type | Description |
|--------|------|-------------|
| `input.attach(target?)` | `void` | Register listeners. Defaults to `window`. |
| `input.detach()` | `void` | Remove listeners. Call in `onDestroy`. |
| `input.flush()` | `void` | Advance frame state. Call once before reading. |
| `input.mouseX` | `number` | Mouse cursor X in client coordinates. |
| `input.mouseY` | `number` | Mouse cursor Y in client coordinates. |
| `input.isMouseDown(btn?)` | `boolean` | True while left button (0) or specified button is held. |
| `input.isMousePressed(btn?)` | `boolean` | True on the single frame a click begins. |
| `input.touches` | `ReadonlyArray<TouchPoint>` | All currently active touch points. |
| `input.primaryTouch` | `TouchPoint \| undefined` | Lowest-id active touch, or undefined. |
| `input.touchCount` | `number` | Number of active touches. |
| `input.isTouchStarted(id?)` | `boolean` | True if a touch started this frame (optionally by id). |
| `input.isTouchEnded(id?)` | `boolean` | True if a touch ended this frame (optionally by id). |

## Notes

- Always call `input.flush()` at the start of `onUpdate` — without it, `isMousePressed` and `isKeyPressed` will fire on every frame instead of just once.
- `input.mouseX/mouseY` report the most recent mouse position in `clientX/clientY` space — transform to canvas coordinates if your canvas is scaled.
- For keyboard input, see `input.isKeyDown(code)`, `input.isKeyPressed(code)`, `input.isKeyReleased(code)` — they take `KeyboardEvent.code` values (`'Space'`, `'ArrowLeft'`, `'KeyA'`, etc.).
- For gamepad axis and button input, use `GamepadSystem` alongside `InputSystem`.
