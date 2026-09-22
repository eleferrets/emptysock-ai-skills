# Prefabs and pooling

**Use this when** you're defining reusable entity templates (enemies, bullets, pickups) or dealing with anything that spawns and despawns a lot — bullets, particles, wave-based enemies. If you're hand-writing `scene.spawn('name')` + a pile of `.add()` calls for the same shape more than once, this is the file that saves you from that.

---

## Prefabs: a named template, not a live scene node

```typescript
import { definePrefab } from '@emptysock/engine'

const Physical = definePrefab('Physical', [{ def: Transform }, { def: PhysicsBody }])
const Enemy = definePrefab('Enemy', [{ def: Health, overrides: { max: 50 } }], { extends: [Physical] })

const goblin = scene.spawn(Enemy, { x: 100, y: 200 })
```

There's no live parent/child scene tree here — `extends` just flattens `Physical`'s components onto `Enemy`'s own list at spawn time, depth-first. If two prefabs in the chain both touch the same component, the more specific one (the prefab actually being spawned, and anything closer to it in the `extends` chain) wins the override, which is what lets `Enemy` layer its own `Health.max` over whatever `Physical` set.

**Prop matching**: the second argument to `spawn` is a flat prop bag, applied on top of each component's own defaults/overrides. A key matches by field name — `{ x: 100 }` overrides `x` on *every* component in the prefab that happens to have an `x` field (typically just `Transform`, but be aware of the "every" if you've got two components sharing a field name). A prop that matches nothing on any component in the prefab logs a console warning naming the unmatched field — that's almost always a typo, not something to silently ignore.

## Pooling folds into spawn/destroy — there's no separate pool class

```typescript
const bullet = scene.spawn(BulletPrefab, { x, y, vx, vy }, { pool: true })
// later:
scene.destroy(bullet)   // returns it to BulletPrefab's pool instead of deallocating
```

Game code never branches on whether an entity happened to be pooled — `scene.destroy()` is the exact same call either way, and the engine decides internally what actually happens. Spawning the same prefab again with `{ pool: true }` reuses an idle pooled entity if one's available (its components get reset to fresh defaults + your new overrides) instead of allocating a new bitECS entity.

**The one gotcha worth memorizing:** a pooled-and-destroyed entity's `.isAlive` reads `true`, not `false`. This is deliberate — the entity's underlying id is deliberately *not* released back to the engine's normal id-recycling, specifically so an unrelated `spawn()` elsewhere in the scene can't accidentally claim that id before this prefab's own pool gets to reuse it. The practical effect for you: game code holding an old handle to a pooled-and-destroyed entity sees `.get()` return `undefined` for everything (no components are attached), which behaves like "destroyed" for basically every purpose — just don't write a check that branches on `isAlive` to detect "was this thing destroyed" if pooling is in play. Check `.has()`/`.get()` on the specific component you actually care about instead.

## When to reach for pooling vs. plain spawn/destroy

Pool anything that spawns and despawns at a meaningful rate — bullets, hit-effect particles, wave enemies in a horde mode. Skip it for anything that's created once and lives for a whole scene (the player, a boss, level geometry) — pooling a thing you never destroy just complicates the code for nothing.

## JSON-authored prefabs and typed props

Hand-authored prefabs via `definePrefab<T>(...)` can pass an explicit type parameter for typed `props` at the call site. Prefabs authored as `.prefab.json` files in the IDE get their typing for free from the toolchain's codegen (`generatePrefabTypes`), which reads the project's real registered component defs and emits a `declare const SomePrefab: PrefabDef<{...}>` per prefab — that's what actually drives autocomplete on `scene.spawn(SomePrefab, props)` for a JSON-authored prefab. You don't need to do anything to get this beyond having the toolchain build step run.
