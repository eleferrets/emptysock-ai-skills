# @emptysock/network — multiplayer via Colyseus

**Use this when** you're adding multiplayer to a game, replicating component fields across clients, or debugging why a networked value isn't syncing. Not needed for local-only games — nothing in the core engine imports this package, and a single-player game pays nothing for it.

---

## Mark fields, don't wrap components

```typescript
import { defineComponent } from '@emptysock/engine'
import { networked } from '@emptysock/network'

const Position = networked(
  defineComponent('Position', () => ({ x: 0, y: 0 })),
  ['x', 'y'],
)
```

`networked(componentDef, fields)` returns the same def unchanged, so it composes right into a `defineComponent(...)` chain — no `defineNetworkedComponent` wrapper to learn, no separate registry to keep in sync with `ComponentRegistry`. Call it once, on both client and server, keyed by the same `componentName` string the rest of the engine already uses for that component.

## The one rule that actually matters: reassign, don't mutate

`NetworkSystem.sync()` decides what to send by comparing each networked field's *current* value against the last value it sent, with strict equality. For a primitive field (a number, a string) that's exactly "did this change." For an object or array field, mutating it in place doesn't change which reference is stored — `component.inventory.push(item)` never trips the check, because `component.inventory` is still the same array reference it was before. If a networked field ever holds an object or array, always assign a whole new value:

```typescript
component.inventory = [...component.inventory, item]   // this replicates
component.inventory.push(item)                          // this silently doesn't
```

This isn't a bug waiting to be fixed — it's a documented tradeoff for keeping the dirty-check cheap, and the fix (deep equality, or watching mutations some other way) is a bigger design call than a field-marking helper should make on its own. Just know the rule and you'll never hit it.

## Sync cadence

`NetworkSystem.sync()` is meant to be called at a low, fixed cadence — a handful of times a second, not every frame. This package is built for "replicate state a handful of times a second to a handful of clients," not for competing with `scene.each()`'s no-proxy hot path. If your game needs tighter netcode than that (rollback, client-side prediction at 60Hz), you're past what this package is trying to solve and should look at Colyseus directly, or budget real engineering time for it — the engine explicitly didn't try to write rollback netcode itself (see the engine's own design notes on "don't reinvent solved problems").

## Bridging entities to network ids

Colyseus room state has no idea what a local entity id even is, and entity ids are only unique within one process's ECS world anyway. `NetworkEntityMap` bridges a Colyseus network id (typically `room.sessionId` for a player, or a synthetic key for a server-spawned entity) to a local `Entity` handle:

```typescript
import { NetworkEntityMap, NetworkSystem } from '@emptysock/network'

const entityMap = new NetworkEntityMap()
const netSystem = new NetworkSystem(room, scene, entityMap, { componentDefs: [Position, Health] })

// low, fixed cadence — not every frame
setInterval(() => netSystem.sync(), 100)
```

`NetworkSystem` never reaches past `entity.get(Component)` to touch a networked field — it doesn't know bitECS exists, and it never will need to. It also reconciles stale mappings automatically: every `sync()` call first checks each tracked entity's already-public `.isAlive` and drops the mapping for anything that's gone. This matters because a locally `scene.destroy()`-ed entity has no way to push a notification into `NetworkEntityMap` on its own — without this reconciliation, a destroyed entity's recycled id could get silently aliased to a completely unrelated entity on the next sync. If you need tighter reconciliation than `sync()`'s own cadence, call `netSystem.reconcile()` directly — it's public for exactly that.

## What this package will never do

It will never import bitECS internals, and the core engine will never import this package. If you're tempted to reach past `entity.get()`/`.add()` from network code to "go faster," that's the wrong instinct here — the whole design bet is that a handful of syncs a second through the ordinary component API is plenty fast enough for what multiplayer games actually need most of the time.
