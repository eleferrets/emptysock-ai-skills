# SaveSystem — generic component save/load

**Use this when** you're implementing save slots, save file migration, or asking "how do I persist an entity's state." Not for `LocalisationSystem` (see `skills/07-save-localisation.md`, which also covers the standalone `SaveSystem` still used for systems outside the entity/component core).

---

## The whole point: zero per-component save code

`SaveSystem` is bound to a scene and an explicit list of "save-aware" component defs. It reads every entity's listed components straight through the same `get()`/`has()` machinery you already use, and serializes them generically:

```typescript
import { SaveSystem } from '@emptysock/engine'

const save = new SaveSystem(scene, [Transform, Health, Inventory])
await save.save('slot-1')
await save.load('slot-1')
```

There's no `Health.serialize()` to write. This works precisely because component fields are constrained to `Serializable` data in the first place (see `skills/32-ecs-core.md`) — anything a `defineComponent` shape can legally hold, `SaveSystem` can already snapshot and restore without being told how.

The explicit component list is deliberate, not an oversight — it mirrors `scene.each(...)`'s own design (no global "every component that has ever existed" registry anywhere in the engine). It also means purely-visual or purely-derived runtime state doesn't accidentally end up in a save file just because it happens to be a component.

## Storage backend

With no adapter given, saves live in memory only — nothing persists across a restart. That's exactly what you want in tests and the headless harness. A real game gets a real adapter injected by its host: IndexedDB in the browser preview, Tauri's fs plugin on desktop. You never write that adapter yourself inside game logic; it's handed in:

```typescript
new SaveSystem(scene, components, { adapter: myAdapter, keyPrefix: 'myGame_' })
```

## Versioning and migrations

Every component carries a schema version (default `1`, bump it explicitly):

```typescript
const Inventory = defineComponent('Inventory', () => ({ items: [] }), { version: 2 })

save.registerMigration('Inventory', (oldData, oldVersion) => {
  // oldVersion tells you what shape oldData is actually in
  return { items: oldData.items ?? [], gold: 0 }
})
```

On load, a saved component instance stamped with an older version than the currently-registered def runs its migration if one's registered. If none is registered, that one component's data is dropped for that one entity, with a console warning — the rest of the save still loads. A shape mismatch on `Inventory` never takes down a save that also has `Transform` and `Health` in it. Write it to return exactly the *current* shape — whatever it returns is trusted as-is and handed straight to the entity.

## Other useful calls

```typescript
await save.hasSave('slot-1')      // boolean
await save.listSlots()            // string[]
await save.deleteSave('slot-1')
```

## A note on scope

`SaveSystem` only knows about the component defs you construct it with, for one specific `Scene`. If your game has multiple scenes with save-relevant state, either construct one `SaveSystem` per scene with the components that scene cares about, or route through a shared save orchestration layer in your own game code — the engine doesn't assume a single global save shape for you.
