# Touch & Pointer Input

**Use this when** you're reading keyboard, mouse, touch or pen state. The normal path is the game-owned `InputManager` (`ctx.input` / `game.input`): named actions for keys and gamepads, plus per-frame frozen pointer, gesture and wheel lists. `InputSystem` (keyboard) and `PointerSystem` are the layers underneath it (both exported, rarely needed directly). For gestures and wheel detail see `skills/25-pointer-system.md`.

All pointer data (mouse, touch, pen) comes through `InputManager`; there is no separate `touches` API.

## Setup

```typescript
import { defineScene, InputManager } from '@emptysock/engine'

// Host bootstrap (browser/Tauri), once:
const input = new InputManager({
  jump: [{ kind: 'key', code: 'Space' }],
})
input.attach()   // starts keyboard + pointer listeners on window; Game never calls this itself

// In a scene, use the game's own manager from the lifecycle context:
export const GameScene = defineScene({
  onLoad(scene, ctx) { /* ctx.input is the same InputManager the game owns */ },
  onUpdate(dt) { /* read input here; Game froze this frame's snapshot already */ },
})
```

`Game.update()` calls `input.snapshot()` once per frame before your `onUpdate`, so every read in a frame sees the same state.

## Reading pointers (mouse, touch, pen)

```typescript
import type { PointerState } from '@emptysock/engine'

const pointers: ReadonlyArray<PointerState> = input.pointers

for (const p of pointers) {
  // p: { id, x, y, dx, dy, startX, startY, startTime, pointerType, isPrimary, buttons }
  console.log(p.id, p.x, p.y, p.pointerType)
}

const primary = pointers.find((p) => p.isPrimary)   // mouse, or the first active touch
const leftButtonHeld = primary !== undefined && (primary.buttons & 1) === 1
```

Coordinates are in the same space the pointer events deliver (client pixels); convert to canvas or world coordinates yourself if the canvas is scaled (see `skills/24-viewport-system.md`, `CameraSystem.screenToWorld`).

## Virtual buttons via pointer position

```typescript
const HALF = GAME_WIDTH / 2
const p = input.pointers.find((q) => q.isPrimary)
const down = p !== undefined && (p.buttons & 1) === 1
const movingLeft  = down && p.x < HALF
const movingRight = down && p.x >= HALF
```

## Multi-touch

```typescript
for (const touch of input.pointers) {
  if (touch.pointerType !== 'touch') continue
  console.log(touch.id, touch.x, touch.y, touch.dx, touch.dy)   // dx/dy: delta since last move
}
```

There is no "touch started/ended this frame" helper. Detect new or lifted touches by comparing `input.pointers` ids with the previous frame's, or subscribe through `PointerSystem.onPointerDown` / `onPointerUp` (see skill 25).

## Keyboard actions

```typescript
input.wasPressed('jump')     // pressed this frame
input.isDown('jump')         // held
input.wasReleased('jump')    // released this frame
input.keyboard.isDown('KeyA')      // raw physical key (KeyboardEvent.code)
input.keyboard.isCharDown('a')     // layout-aware: the key that types "a"
```

## API reference (`InputManager`)

| Member | Type | Description |
|--------|------|-------------|
| `attach(target?)` | `void` | Start listening to real device events (defaults to `window`) |
| `detach()` | `void` | Stop listening |
| `snapshot()` | `void` | Freeze this frame's state (Game calls it) |
| `isDown(action)` / `wasPressed(action)` / `wasReleased(action)` | `boolean` | Action queries |
| `keyboard` | `KeyboardSnapshot` | `isDown(code)`, `isCharDown(ch)` |
| `gamepad(index)` | `GamepadSnapshot` | `connected`, `isButtonDown(i)`, `axis(i)` |
| `pointers` | `ReadonlyArray<PointerState>` | Active pointers |
| `gestures` | `ReadonlyArray<Gesture>` | Gestures since the previous snapshot |
| `wheelEvents` | `ReadonlyArray<WheelEventInfo>` | Wheel events since the previous snapshot |

Rebinding: `setActions`, `bindAction`, `rebind`, `addBinding`, `unbind`, `resetToDefaults`, `saveBindings`/`loadBindings`, `captureNext`, `rebindByCapture` (see `skills/28-accessibility-debugging.md`).

## Raw `InputSystem`

`InputSystem` is the keyboard-only layer `InputManager` wraps. It is exported from `@emptysock/engine` (with the `KeyState` type), but game code normally reads keys through `input.keyboard` / actions instead. Its edge semantics are the same as the manager's: pressed/released are true for exactly one frame.

## Notes

- Keys are `KeyboardEvent.code` values (`'Space'`, `'ArrowLeft'`, `'KeyA'`).
- For gamepad input use bindings (`{ kind: 'gamepadButton' | 'gamepadAxis' }`) or `GamepadSystem` directly.
