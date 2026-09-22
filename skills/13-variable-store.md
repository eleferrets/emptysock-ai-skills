# VariableStore

**Use this when** you need global game state that persists across sessions and needs to gate dialogue or map events — numbered variables and switches, the same idea as the RPG Maker MV database if you've used it. `VariableStore` (and its module-level singleton `variableStore`) is backed by `localStorage` and shows up in the IDE's Variables panel automatically.

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
import { SaveSystem, variableStore } from '@emptysock/engine'

// SaveSystem is synchronous — embed a snapshot() call inside a slot's `data` field:
const saves = new SaveSystem()
saves.save('slot1', { scene: 'Map01', data: { vars: variableStore.snapshot() } })

// Restore on load — load() already validates against GameSaveSlot, so no
// separate parse step is needed:
const slot = saves.load('slot1')
if (slot !== null) {
  variableStore.restore(slot.data.vars as ReturnType<typeof variableStore.snapshot>)
}
```

See `skills/07-save-localisation.md` for the full `SaveSystem` API.

---

## Conditional logic — `VariableCondition`

`VariableCondition` is the shared seam that gates conditional dialogue in `VNSystem` and conditional map events in `MapEventSystem` — see `skills/08-story-graph.md` and `skills/15-map-events.md`. You only need to call `evaluateCondition()` directly when building a conditional gate of your own outside those two systems.

```typescript
import { variableStore, evaluateCondition, type VariableCondition } from '@emptysock/engine'

type VariableCondition =
  | { kind: 'switch'; index: number; equals: boolean }
  | { kind: 'variable'; index: number; op: 'eq' | 'neq' | 'gt' | 'gte' | 'lt' | 'lte'; value: number }

variableStore.setSwitch(1, true)
evaluateCondition(variableStore, { kind: 'switch', index: 1, equals: true })  // true

variableStore.setVar(4, 12)
evaluateCondition(variableStore, { kind: 'variable', index: 4, op: 'gte', value: 10 })  // true
```

`VNSystem` and `MapEventSystem` both default to the shared `variableStore` singleton when constructed with no arguments, so a game with one save file needs no extra wiring. Pass a different `VariableStore` instance to either constructor when isolation is needed instead — per-save-slot state, or a clean store in a test:

```typescript
import { VariableStore, VNSystem, MapEventSystem } from '@emptysock/engine'

const isolatedStore = new VariableStore()
const vn = new VNSystem(isolatedStore)
const events = new MapEventSystem(isolatedStore)
```

---

## IDE panel

The **Variables** panel (enable via **Module → Variables**) shows all named variables and switches in an editable table. Changes in the panel are written directly to `ideStore` and reflected at runtime. The panel state is saved to `emptysock.project.json` with the project.

---

## Rules

- `variableStore` is the shared default singleton — use it unless you genuinely need an isolated store (per-save-slot state, tests), in which case construct your own `VariableStore` and pass it explicitly to `VNSystem` / `MapEventSystem`.
- Call `load()` once in your entry scene's `onLoad()`, not on every scene transition.
- Call `save()` after a meaningful state change (checkpoint, item collected, boss defeated) — not every frame. Nobody needs gold saved 60 times a second.
- Indices are 1-based. Index 0 works but quietly aliases to slot 1 internally, so stick to 1+ to avoid confusing yourself later.
