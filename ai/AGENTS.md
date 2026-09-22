# EmptySock — Agent Reference

Vendor-agnostic instructions for any AI agent working with the EmptySock game engine.
Use this as a system prompt prepend, context file, or project instruction.

---

## What is EmptySock?

EmptySock is a TypeScript game engine for 2D games (with optional 3D) that exports to
Web, Windows, macOS, Linux, Android, iOS, and Raspberry Pi.

The engine package is `@emptysock/engine`. It wraps a bitECS-based entity/component
core, PixiJS (rendering), Rapier2D/3D WASM (physics), and Howler.js (audio) behind a
clean, fully-typed API. Agents never import those underlying libraries directly — only
`@emptysock/engine`.

Four optional companion packages exist alongside it and are never pulled in by the
core engine automatically: `@emptysock/network` (Colyseus multiplayer),
`@emptysock/vn` (Story Graph / VNSystem), `@emptysock/battle` (turn-based battles),
`@emptysock/tilemap` (Tilemap + NavMeshSystem). Import one only when the game actually
needs it.

---

## Non-negotiable rules

1. **TypeScript strict mode when writing TS.** Zero `any`. Zero `!` non-null
   assertions. External data (JSON files, save data, network payloads) parsed through
   Zod before use. Explicit return types on every function. JavaScript is equally
   first-class here — both languages compile/run against the identical API, there's
   no "simplified" surface for one or the other.

2. **No direct library imports.** Never import `pixi.js`, `@dimforge/rapier2d`/`3d`,
   or `howler`. The engine re-exports what you need.

3. **No DOM access in game logic.** Never call `document.querySelector` or similar
   directly inside scene/entity code.

4. **No `setTimeout`/`setInterval` in game logic.** Use the engine's own timer/tween
   API so it respects pause and scene teardown.

5. **`onUpdate` must not be async.** In TypeScript this is an actual compile error —
   `onUpdate` is typed `(dt: number) => void`, and returning a `Promise` doesn't
   satisfy that. The game loop calls it synchronously and never awaits the result, so
   anything after an `await` inside would run detached from the frame budget, with no
   way for the engine to catch an error thrown there. Use `entity.startCoroutine(...)`
   for anything that spans multiple frames. In JavaScript there's no compile-time
   catch, but the engine logs a loud runtime warning if `onUpdate` ever returns
   something Promise-shaped.

6. **Destroy what you spawn.** `scene.destroy(entity)` when an entity is done —
   whether or not it was spawned pooled, this is the one call, and the engine decides
   internally whether that means real deallocation or returning it to a pool.

7. **Validate external/saved data.** Never cast `unknown` straight to a type. Parse
   it through a schema or a real guard first — saved data and imported files can be
   stale, hand-edited, or from an older version of your game.

---

## Core object model

Entities are lightweight handles, not classes to subclass. Components are plain data
defined once with `defineComponent`, keyed by a stable name string (this is what
survives a hot-reload, where the module re-evaluating hands you a brand-new object
reference for "the same" component):

```typescript
const Health = defineComponent('Health', () => ({ current: 100, max: 100 }))

const goblin = scene.spawn('Goblin')
goblin.add(Health, { current: 30 })
goblin.get(Health)          // proxy onto the live data, or undefined if absent/destroyed
goblin.has(Health)          // boolean
scene.destroy(goblin)
```

For touching many entities at once, `scene.each(...)` is the bulk path — same
components, same object, just skips the per-entity proxy allocation `.get()` does:

```typescript
scene.each(Transform, PhysicsBody, (transform, body, entity) => {
  transform.x += body.velocity.x * dt
})
```

Component fields must be JSON-serializable (`Serializable`) — no functions, no class
instances. A component that needs a callback-shaped property (collision handlers,
for instance) keeps that callback in a side-table behind a small wrapper instead —
you still assign it like a normal property, it's just not stored as component data.

## Scene lifecycle

```typescript
const GameScene = defineScene({
  onLoad(scene, { actors, physics, input, audio }) { /* build the world */ },
  onUpdate(dt) { /* every frame */ },
  onUnload(scene) { /* teardown before actors/physics are destroyed */ },
})

const game = new Game()
await game.loadScene(GameScene)
```

`Game` creates and destroys that scene's `ActorSystem`/`PhysicsSystem` for you — you
never construct those yourself unless you explicitly pass
`{ manageLifecycle: false }`. `game.loadOverlay(...)` stacks an independently
lifecycled scene (HUD, pause menu) on top, surviving a main-scene reload underneath
it. `game.services` is a typed registry for process-global state (score tracking,
analytics, anything that would've been a singleton) — register once, `.get()`
anywhere, no ambient globals.

## Physics

Plain-language properties over Rapier — `velocity`, `type: 'dynamic' | 'static' |
'kinematic'`, `shape`. Collision callbacks are a direct property assignment:

```typescript
const body = player.get(PhysicsBody)
body.onCollisionEnter = (other, contact) => { /* ... */ }
```

Bodies collide with each other by default — collision groups are opt-in tuning, not
a setup requirement.

## Prefabs and pooling

```typescript
const Enemy = definePrefab('Enemy', [{ def: Health, overrides: { max: 50 } }], { extends: [Physical] })
scene.spawn(Enemy, { x: 100 }, { pool: true })
```

Pooling is folded into `spawn`/`destroy` — no separate pool class. One thing worth
knowing: a pooled-and-destroyed entity's `isAlive` still reads `true` (the id is held
for that prefab's own reuse, not released generally) — check `.has()`/`.get()` on
specific components to tell if something's "really" gone, not `isAlive`, for a pooled
entity.

## Saving

```typescript
const save = new SaveSystem(scene, [Transform, Health, Inventory])
await save.save('slot-1')
await save.load('slot-1')
```

Zero per-component save code required — any component built from serializable fields
saves and loads generically. Bump `defineComponent(name, defaults, { version: 2 })`
and register a migration (`save.registerMigration(name, fn)`) when a component's
shape changes; an entity whose saved data has no migration for the version mismatch
just drops that one component's data (with a warning), the rest of the load proceeds.

---

## Commit format

```
feat(player): add wall-jump mechanic
fix(physics): character tunnels through thin platform at high speed
refactor(enemy): extract patrol logic into PatrolComponent
```

---

## Further reference

- `ai/api-reference.json` — full machine-readable API.
- `skills/` — one topic-scoped guide per system.
