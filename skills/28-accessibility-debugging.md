# Accessibility & Debugging

**Use this when** you need remappable controls, a debug console you can actually ship, or accessibility settings like text scale and colourblind simulation. Covers three related pieces: `InputBindings` (control remapping), `DebugOverlaySystem` (a shippable FPS/console overlay), and the accessibility primitives (`accessibilitySettings.textScale`, `PostProcessSystem`'s `'colourblind'` layer filter).

---

## InputBindings — remappable controls

Query named actions instead of raw key codes, and rebinding becomes a settings-menu problem instead of a "rewrite half the gameplay code" problem:

```typescript
import { InputBindings, InputSystem, GamepadSystem, type ActionMap } from '@emptysock/engine'

const defaults: ActionMap = {
  jump: [{ kind: 'key', code: 'Space' }, { kind: 'gamepadButton', index: 0 }],
  left: [{ kind: 'key', code: 'ArrowLeft' }],
  right: [{ kind: 'key', code: 'ArrowRight' }],
}

const input = new InputSystem()
const gamepad = new GamepadSystem()
const bindings = new InputBindings(input, defaults, gamepad)

override onUpdate(dt: number): void {
  input.flush()
  gamepad.update()
  if (bindings.isActionActive('jump') && controller.isGrounded()) jump()
  const h = bindings.isActionActive('right') ? 1 : bindings.isActionActive('left') ? -1 : 0
}
```

Rebind from a settings menu:

```typescript
bindings.rebind('jump', [{ kind: 'key', code: 'KeyZ' }])
bindings.resetToDefaults()
```

### Persisting bindings

`InputBindings` persists through `SaveSystem`'s generic schema API, not the default save-slot shape:

```typescript
import { createBindingsSaveSystem } from '@emptysock/engine'

const bindingsSave = createBindingsSaveSystem()   // SaveSystem<BindingsSaveSlot>
bindings.save(bindingsSave)
bindings.load(bindingsSave)   // returns false if nothing was persisted yet
```

---

## DebugOverlaySystem — shippable in-game debug console

Off by default. Gate `enable()` behind whatever dev flag your game already defines (a query param, a build-time constant) so it never accidentally ships live:

```typescript
import { DebugOverlaySystem } from '@emptysock/engine'

const debugOverlay = new DebugOverlaySystem()
this.uiSystem.add(debugOverlay.root)   // draws through the normal Widget tree — add it above other UI

if (isDevBuild) debugOverlay.enable()

debugOverlay.registerCommand('give', (args) => {
  giveItem(args[0])
  return `gave ${args[0]}`
})

override onUpdate(dt: number): void {
  debugOverlay.update(dt, this.entityCount)
}
```

`log()`/`warn()`/`logError()` append to the on-screen console (capped at 200 lines, 8 shown at a time). `runCommand('give sword')` parses and dispatches to a registered handler; `help` and `clear` are built in.

---

## accessibilitySettings.textScale

A global text-scale multiplier every `LabelWidget` reads at render time. One settings-menu slider, and every label already on screen, in every scene, follows along:

```typescript
import { accessibilitySettings } from '@emptysock/engine'

// In a settings panel:
accessibilitySettings.textScale = 1.5   // clamped to [0.5, 3]
accessibilitySettings.reset()           // back to 1
```

---

## Colourblind simulation filter

`PostProcessSystem.setLayerFilter()` accepts a `'colourblind'` filter type that simulates protanopia, deuteranopia, or tritanopia using the standard Brettel/Viénot/Machado matrices:

```typescript
postProcess.setLayerFilter('ui', { type: 'colourblind', mode: 'deuteranopia' })
```

**This is a simulation, not a correction.** It shows a non-colourblind player what a colourblind player sees, for design review — it doesn't make colours any easier for an actual colourblind player to tell apart. A real daltonisation/correction filter is a separate feature that doesn't exist yet; don't describe this one as that.

---

## Rules

| Wrong | Right |
|---|---|
| Checking `input.isKeyDown('Space')` directly in gameplay code | Check `bindings.isActionActive('jump')` so remapping works everywhere at once |
| Persisting `InputBindings` through the default `SaveSystem` slot | Use `createBindingsSaveSystem()` / `BindingsSaveSlotSchema` — a different schema entirely |
| Shipping `DebugOverlaySystem` always enabled | Gate `enable()` behind a dev flag the game itself defines |
| Scaling text by hand, per widget, for accessibility | Set `accessibilitySettings.textScale` once — every `LabelWidget` reads it |
| Calling the `'colourblind'` filter a correction/fix for colourblind players | It's a preview for sighted designers, not a correction |
