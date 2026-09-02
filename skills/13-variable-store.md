# VariableStore

`VariableStore` (and the module-level singleton `variableStore`) provides numbered integer variables and boolean switches that persist across sessions, mirroring the RPG Maker MV database concept. It is backed by `localStorage` and integrates with the IDE's Variables panel.

---

## Quick start

```typescript
import { variableStore } from '@emptysock/engine'

// Load saved state on scene start
variableStore.load()

// Read and write variables (1-indexed integers)
variableStore.setVar(1, 0)           // initialize gold to 0
variableStore.setVarName(1, 'Gold')  // optional display name
variableStore.setVar(1, variableStore.getVar(1) + 100)  // add 100 gold

// Read and write switches (1-indexed booleans)
variableStore.setSwitchName(1, 'Boss Defeated')
variableStore.setSwitch(1, true)

// Persist after a change
variableStore.save()
```

---

## API

### Variables (integer slots, 1–1000)

```typescript
variableStore.getVar(index: number): number
variableStore.setVar(index: number, value: number): void   // floored to integer
variableStore.getVarName(index: number): string            // '' if unnamed
variableStore.setVarName(index: number, name: string): void
```

### Switches (boolean slots, 1–1000)

```typescript
variableStore.getSwitch(index: number): boolean
variableStore.setSwitch(index: number, value: boolean): void
variableStore.getSwitchName(index: number): string
variableStore.setSwitchName(index: number, name: string): void
```

### Persistence

```typescript
variableStore.save(): void     // write to localStorage
variableStore.load(): void     // read from localStorage (silently no-ops if empty)
variableStore.reset(): void    // clear all variables and switches
```

### Save file integration

```typescript
// Include in a SaveSystem slot
const data = variableStore.snapshot()
saveSystem.save('slot1', { scene: 'Map01', data: { vars: data }, timestamp: Date.now(), playtime: 0 })

// Restore on load
const slot = saveSystem.load('slot1')
if (slot !== null) {
  variableStore.restore(slot.data['vars'] as VariableStoreData)
}
```

---

## IDE panel

The **Variables** panel (enable via **Module → Variables**) shows all named variables and switches in an editable table. Changes in the panel are written directly to `ideStore` and reflected at runtime. The panel state is saved to `emptysock.project.json` with the project.

---

## Rules

- `variableStore` is a process-global singleton — do not create new `VariableStore` instances per scene.
- Call `load()` once in your entry scene's `onLoad()`, not on every scene transition.
- Call `save()` after any meaningful state change (checkpoint, item collected, boss defeated) — do not call every frame.
- Indices are 1-based. Index 0 is accepted but treated as the same slot as 1 internally; use 1+ for clarity.
