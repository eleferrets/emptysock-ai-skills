# Touch Input

Touch events are exposed through the static `Input` class alongside keyboard and pointer input. No setup or attach call needed — the engine registers touch listeners automatically.

## Import

```typescript
import { Input } from '@emptysock/engine'
```

## Reading touch state (in onUpdate)

```typescript
// All active touches:
const touches = Input.touches   // Touch[]

if (touches.length > 0) {
  const primary = touches[0]
  console.log(primary.x, primary.y)
}

// Pointer (mouse or first touch — works for both):
const pos      = Input.pointer.position   // { x: number, y: number }
const isHeld   = Input.pointer.isDown     // true while finger is down
const tapped   = Input.pointer.isPressed  // true on the frame a tap starts
```

## Virtual buttons via pointer

Map screen regions to game actions using the pointer position:

```typescript
override onUpdate(dt: number): void {
  // Simple left/right virtual d-pad split at screen centre:
  const HALF = GAME_WIDTH / 2
  const movingLeft  = Input.pointer.isDown && Input.pointer.position.x < HALF
  const movingRight = Input.pointer.isDown && Input.pointer.position.x >= HALF

  if (movingLeft)  { /* move left */ }
  if (movingRight) { /* move right */ }
}
```

## Multi-touch example

```typescript
override onUpdate(dt: number): void {
  for (const touch of Input.touches) {
    // Each touch has .x and .y in canvas coordinates.
    // Use their positions to drive virtual joysticks, buttons, etc.
  }
}
```

## API reference

| Member | Type | Description |
|--------|------|-------------|
| `Input.touches` | `Touch[]` | All currently active touch points. |
| `Input.pointer.position` | `{ x: number, y: number }` | Mouse cursor or first touch position. |
| `Input.pointer.isDown` | `boolean` | True while the primary pointer button or finger is held. |
| `Input.pointer.isPressed` | `boolean` | True on the single frame a tap or click begins. |

## Notes

- `Input` is a static class — never instantiate it.
- `Input.pointer` unifies mouse and first touch into one property. For most games this is all you need.
- Use `Input.touches` only when you need multi-touch (two-finger gestures, multi-touch controls).
- Reading `Input.touches.length` is safe when there are no touches — it returns an empty array, not null.
