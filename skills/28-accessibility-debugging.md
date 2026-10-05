# Accessibility & Debugging

**Use this when** you need remappable controls, a debug console you can actually ship, or accessibility settings like text scale and colourblind simulation. Covers three related pieces: `InputManager` remappable actions (control remapping), `DebugOverlaySystem` (a shippable FPS/console overlay), and the accessibility primitives (a text-scale pattern and `PostProcessSystem`'s `'colourblind'` layer filter).

---

## InputManager: remappable controls

`game.input` (an `InputManager`, also `ctx.input` in a scene) is the one input system, and named actions are its default mode: query an action instead of a raw key code and rebinding becomes a settings-menu problem, not a "rewrite half the gameplay code" problem. There is no separate bindings class; remapping, persistence and the raw-device escape hatches all live on `InputManager`.

```typescript
import { InputManager, type ActionMap } from '@emptysock/engine'

const defaults: ActionMap = {
  jump:  [{ kind: 'key', code: 'Space' }, { kind: 'gamepadButton', index: 0 }],
  left:  [{ kind: 'key', code: 'ArrowLeft' }, { kind: 'gamepadAxis', axis: 0, threshold: -0.5 }],
  right: [{ kind: 'key', code: 'ArrowRight' }, { kind: 'gamepadAxis', axis: 0, threshold: 0.5 }],
}
game.input.setActions(defaults)   // game.input starts with an empty action map

// in onUpdate: state is a frozen per-frame snapshot, taken by Game.update()
const input = game.input   // or ctx.input inside a scene
if (input.wasPressed('jump') && grounded) jump()
const h = input.isDown('right') ? 1 : input.isDown('left') ? -1 : 0
```

- Any active binding makes the action active (`isDown`). `wasPressed` / `wasReleased` are true for exactly one frame; there is no separate `update()` call to make.
- `resetToDefaults()` restores the map an `InputManager` was *constructed* with. `game.input` is constructed empty, so keep your own `defaults` constant and reset with `setActions(defaults)`; call `resetToDefaults()` only on a manager you built with `new InputManager(defaults)`.
- A gamepad axis binding's `threshold` sign matters: negative means "at or below", positive means "at or above".
- Key bindings use the physical key position (`KeyboardEvent.code`: `'KeyW'`, `'Space'`), so WASD stays where the fingers expect on AZERTY and Dvorak. Label keys in a settings UI with the browser's layout map where available.

Rebind from a settings menu:

```typescript
input.rebind('jump', [{ kind: 'key', code: 'KeyZ' }])   // replace an action's bindings
input.addBinding('jump', { kind: 'gamepadButton', index: 1 })   // append (duplicates ignored)
input.unbind('jump', { kind: 'key', code: 'KeyZ' })      // remove one; omit the second argument to remove the action
input.setActions(defaults)                               // reset to your own defaults constant
```

### Persisting bindings

Bindings persist through any `StorageAdapter` (the same interface `SaveSystem` takes), not through a save slot:

```typescript
import { INPUT_BINDINGS_STORAGE_KEY } from '@emptysock/engine'

await input.saveBindings(storage)              // key defaults to 'emptysock_input_bindings'
const restored = await input.loadBindings(storage)   // false if nothing stored or the data is corrupt; bindings untouched
```

`loadBindings` validates the stored shape before applying it, so stale or hand-edited data can never leave you with half a control scheme.

Raw escape hatches read the same frozen snapshot: `input.keyboard.isDown('KeyW')`, `input.gamepad(0)`, `input.pointers`, `input.gestures`, `input.wheelEvents`.

---

## DebugOverlaySystem — shippable in-game debug console

Off by default. It is built from widget entities, so it needs the scene and `WidgetTree` you already use for UI (see `skills/11-ui-system.md`). Gate `enable()` behind whatever dev flag your game already defines (a query param, a build-time constant) so it never accidentally ships live:

```typescript
import { DebugOverlaySystem } from '@emptysock/engine'

const debugOverlay = new DebugOverlaySystem(scene, tree)   // scene: Scene, tree: WidgetTree
// debugOverlay.root is the overlay's root widget entity

if (isDevBuild) debugOverlay.enable()      // also disable() / toggle() / enabled

debugOverlay.registerCommand('give', (args) => {
  giveItem(args[0])
  return `gave ${args[0]}`
})

// each frame, from onUpdate:
debugOverlay.update(dt, entityCount)
```

`log()`/`warn()`/`logError()` append to the on-screen console (capped at 200 lines, 8 shown at a time; read back with `history`). `runCommand('give sword')` parses and dispatches to a registered handler and returns its output; `help` and `clear` are built in. Also `unregisterCommand(name)` and `commandNames`.

---

## Text scale

`@emptysock/engine` exports a global text-scale setting: the `accessibilitySettings` singleton (class `AccessibilitySettings`) with `textScale`, clamped to [0.5, 3], and `reset()`. It is only a shared place to store the value: `UISystem` does not read it, so you apply it yourself when you build or update labels.

```typescript
import { accessibilitySettings } from '@emptysock/engine'

accessibilitySettings.textScale = 1.25   // clamped to [0.5, 3]

function applyTextScale(label: { fontSize: number }, baseSize: number): void {
  label.fontSize = baseSize * accessibilitySettings.textScale
}
```

---

## Colourblind simulation filter

`PostProcessSystem.setLayerFilter()` accepts a `'colourblind'` filter type that simulates protanopia, deuteranopia, or tritanopia using the standard Brettel/Viénot/Machado matrices:

```typescript
postProcess.setLayerFilter('ui', { type: 'colourblind', mode: 'deuteranopia' })   // PostProcessSystem is exported; see skills/21-post-process.md
```

**This is a simulation, not a correction.** It shows a non-colourblind player what a colourblind player sees, for design review — it doesn't make colours any easier for an actual colourblind player to tell apart. A real daltonisation/correction filter is a separate feature that doesn't exist yet; don't describe this one as that.

---

## Rules

| Wrong | Right |
|---|---|
| Checking `input.keyboard.isDown('Space')` directly in gameplay code | Check `input.isDown('jump')` so remapping works everywhere at once |
| Persisting bindings by hand into a `SaveSystem` slot | `input.saveBindings(storage)` / `loadBindings(storage)` |
| Shipping `DebugOverlaySystem` always enabled | Gate `enable()` behind a dev flag the game itself defines |
| Hard-coding font sizes everywhere | Multiply font sizes by a single text-scale setting when you build or update labels |
| Calling the `'colourblind'` filter a correction/fix for colourblind players | It's a preview for sighted designers, not a correction |
