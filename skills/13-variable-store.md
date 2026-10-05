# VariableStore

**Use this when** you need numbered integer variables and boolean switches (the RPG Maker model) that gate dialogue and persist with saves. There is no module-level singleton; the engine owns one `VariableStore` per `Game`, handed to every scene as `ctx.variables`.

---

## Quick start

```typescript
import { defineScene } from '@emptysock/engine'

export const Shop = defineScene({
  onLoad(scene, { variables }) {
    variables.setVarName(1, 'Gold')                 // optional display name
    variables.setVar(1, variables.getVar(1) + 100)  // add 100 gold

    variables.setSwitchName(1, 'Boss Defeated')
    variables.setSwitch(1, true)
  },
})
```

You can also construct one directly: `import { VariableStore } from '@emptysock/engine'; const store = new VariableStore()`.

---

## API

Indices run 1 to 1000 for both kinds and are clamped: 0 or negative values alias to 1, values above 1000 alias to 1000. Use 1-based indices.

```typescript
store.getVar(index: number): number
store.setVar(index: number, value: number): void       // truncated to an integer
store.getVarName(index: number): string                // '' if unnamed
store.setVarName(index: number, name: string): void

store.getSwitch(index: number): boolean
store.setSwitch(index: number, value: boolean): void
store.getSwitchName(index: number): string
store.setSwitchName(index: number, name: string): void

store.save(): void       // write the snapshot to storage (localStorage when available)
store.load(): void       // read it back; silently ignores missing or corrupt data
store.reset(): void      // clear all variables, switches, and names
store.snapshot(): VariableStoreData
store.restore(data: Partial<VariableStoreData>): void
```

### Save file integration

Pass the store to `SaveSystem` and it is written and restored with every slot (see `skills/33-save-system.md`):

```typescript
const save = new SaveSystem(scene, [Transform, Health], { variables })
await save.save('slot-1')
await save.load('slot-1')
```

---

## Conditional logic: `VariableCondition`

`VariableCondition` gates conditional dialogue in `VNSystem` (see `skills/08-story-graph.md`); call `evaluateCondition()` directly when building your own gate.

```typescript
import { evaluateCondition, type VariableCondition } from '@emptysock/engine'

type VariableCondition =
  | { kind: 'switch'; index: number; equals: boolean }
  | { kind: 'variable'; index: number; op: 'eq' | 'neq' | 'gt' | 'gte' | 'lt' | 'lte'; value: number }

store.setSwitch(1, true)
evaluateCondition(store, { kind: 'switch', index: 1, equals: true })  // true

store.setVar(4, 12)
evaluateCondition(store, { kind: 'variable', index: 4, op: 'gte', value: 10 })  // true
```

`VNSystem` takes a `VariableStore` in its constructor; pass `ctx.variables` to share state with the rest of the game, or `new VariableStore()` for an isolated one (per-slot state, tests).

---

## IDE panel

The **Variables** panel (Module menu) shows named variables and switches in an editable table and stores them with the project.

---

## Rules

- Use `ctx.variables` unless you genuinely need an isolated store.
- Call `save()` after a meaningful state change, not every frame. Prefer `SaveSystem` with the `variables` option for save slots.
- Indices are 1-based.
