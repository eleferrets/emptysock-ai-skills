# Touch Input

The InputSystem tracks multi-touch points at frame accuracy with passive event listeners.

## Usage

```typescript
import { InputSystem } from '@emptysock/engine';

const input = new InputSystem();
input.attach(canvasElement);

// In game loop, call flush() at start of each frame:
input.flush();

// Number of active touches:
const count = input.touchCount;

// Primary (first) touch:
const primary = input.primaryTouch; // TouchPoint | undefined
if (primary) {
  console.log(primary.x, primary.y, primary.dx, primary.dy);
}

// All active touches:
for (const touch of input.touches) {
  console.log(touch.id, touch.x, touch.y);
}

// Check if a touch started or ended this frame:
if (input.isTouchStarted()) { /* any touch started */ }
if (input.isTouchEnded(myId)) { /* specific touch ended */ }

// Get by id:
const t = input.getTouch(someId);

// Detach listeners when done:
input.detach(canvasElement);
```

## TouchPoint
```typescript
interface TouchPoint {
  id: number; // browser touch identifier
  x: number;  // current x
  y: number;  // current y
  dx: number; // delta x since last frame
  dy: number; // delta y since last frame
}
```

## Notes
- All touch listeners are registered as `{ passive: true }` for scroll performance.
- `flush()` clears frame-accurate started/ended sets — call once per frame before reading input.
