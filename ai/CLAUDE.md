# EmptySock — Claude Code Instructions

You're building a game with **EmptySock** (`@emptysock/engine`), a TypeScript-first 2D game engine (with optional 3D) that targets web, desktop, mobile, and Raspberry Pi.

Read this file fully before writing any code. It covers the current engine — a bitECS-backed core (call it "v2" if you like, though nobody outside the engine repo needs to care about the version number) plus a handful of v1 systems that never needed rewriting and are still exactly as they were: audio, input devices, rendering, particles, tweens, and the rest. Nothing here is legacy-flavored just because it's old; it's here because it's still the real API.

---

## Engine at a glance

| Layer | Technology |
|---|---|
| ECS core | bitECS, wrapped in a friendly `Entity`/`Scene` handle API |
| 2D Renderer | PixiJS |
| Physics | Rapier2D / Rapier3D (WASM) |
| Audio | Howler.js |
| Shell | Tauri v2 (desktop) |
| Language | TypeScript — strict mode, zero `any` (JavaScript works fine too, see below) |

---

## Absolute rules — never break these

### Types
- Zero `any`. Use `unknown` with a type guard, or Zod if the data's coming from outside the program (a JSON file, save data, a network payload).
- Zero `!` non-null assertions. Use `?.` and `?? defaultValue`.
- Zero `// @ts-ignore`. If you truly can't avoid a type error, `// @ts-expect-error` with a reason.
- `import type` for type-only imports.

### Engine usage
- Never import PixiJS, Rapier, or Howler directly. Everything you need is exported from `@emptysock/engine`.
- Never touch the DOM directly in game logic.
- Never use `setTimeout`/`setInterval` in game logic — use `Timer`/`TweenManager` (whatever's scene-scoped) so it actually respects pause/scene-unload.
- **`onUpdate` cannot be `async`.** This isn't a style rule, it's the type checker: `onUpdate` is typed `(dt: number) => void`, and `defineScene({ async onUpdate() {...} })` is a compile error, on purpose. The game loop calls it synchronously and never awaits it — anything scheduled after an `await` inside would run at a random, frame-budget-detached time, and the engine couldn't catch an error thrown after that point either. For work that spans multiple frames, use `entity.startCoroutine(...)`. Writing plain JavaScript instead of TypeScript? You lose the compile-time catch, but the engine still warns loudly at runtime if `onUpdate` returns something Promise-shaped — it's degraded, not silent.
- Always `scene.destroy(entity)` when an entity is done. If it was spawned with `{ pool: true }`, this returns it to its prefab's pool instead of actually deallocating it — same call either way, you never need to know which happened.

### JavaScript is first-class
You don't have to use TypeScript. Both languages run against the exact same API, and there's no "simple" surface for JS and a "real" one for TS — they're the same objects and the same methods. TypeScript gets you the async-`onUpdate` compile error above and full autocomplete; JavaScript gets a console warning instead of a compile error for that one case, and otherwise loses nothing. Don't apologize for writing JS, and don't add TypeScript ceremony to a JS project just because the docs happen to show `.ts` snippets.

### Commits
- Conventional commit format: `feat(scope): message`, `fix(scope): message`, `refactor(scope): message`.
- Commit after every working addition. Never commit broken code.

---

## The object model: entities are handles, not classes

There's no `class Player extends Entity`. An entity is a lightweight, cheap-to-copy handle — `scene.spawn()` hands you one, and it stays valid until `scene.destroy()`. Behavior comes from attaching components (plain data, defined with `defineComponent`) and reading/writing them; there's no inheritance ceremony to opt into.

```typescript
import { defineComponent, Scene } from '@emptysock/engine'

const Health = defineComponent('Health', () => ({ current: 100, max: 100 }))

const scene = new Scene()
const goblin = scene.spawn('Goblin')
goblin.add(Health, { current: 30 })

const health = goblin.get(Health)   // proxy onto the real component data, or undefined
if (health !== undefined) health.current -= 10   // writes straight through
goblin.has(Health)                                // true
scene.destroy(goblin)
```

`defineComponent(name, createDefaults, options?)` — the `name` string is load-bearing: it's how the engine recognizes "the same" component across a hot-reload, which re-evaluates the module and hands you a brand-new object reference every time. Two components with the same name are treated as one; don't reuse a name for two unrelated shapes. `options.version` (default `1`) and `options.schema` (an optional per-field description used by the IDE's Inspector, and only the Inspector) are both optional and purely additive — most components need neither.

Component fields must be `Serializable` — strings, numbers, booleans, `null`, arrays, and plain objects of the same, no functions or class instances. This is what lets `SaveSystem` serialize any component generically with zero per-component save code (see the save section below). If a component genuinely needs a callback-shaped property (collision callbacks are the standing example), it lives in a side-table behind a small wrapper, not as a component field — `PhysicsBody`'s `onCollisionEnter` etc. work this way and read exactly like an ordinary property assignment even though the callback itself never touches component storage.

**For touching thousands of entities at once**, skip per-entity `.get()` and use `scene.each(...)` — same components, same object, just the bulk-iteration path:

```typescript
scene.each(Transform, PhysicsBody, (transform, body, entity) => {
  transform.x += body.velocity.x * dt
})
```

There's exactly one component system here, not a beginner one and a "real" one underneath it. `each` is just the faster way to touch a lot of entities, the way `Array.forEach` is the faster way to touch a lot of array elements. It reads the raw arrays directly and skips the proxy allocation `.get()` does, which matters once you're iterating hundreds of entities a frame and doesn't matter at all for one enemy's `onUpdate`.

---

## Scenes and lifecycle

A `Scene` owns one ECS world and nothing else — no physics, no actors, no rendering. `Game` is what wires those things to a scene's lifecycle, and it does so automatically:

```typescript
import { Game, defineScene } from '@emptysock/engine'

const GameScene = defineScene({
  onLoad(scene, { actors, physics, input, audio }) {
    // Build the world. actors/physics were created for you — never `new ActorSystem()` yourself.
  },
  onUpdate(dt) {
    // Runs every frame, after physics/actors/collision, before render.
  },
  onUnload(scene) {
    // Called before the engine tears actors/physics down.
  },
})

const game = new Game()
await game.loadScene(GameScene)
// each frame: game.update(dt)
```

**The engine owns everything it creates.** `loadScene` constructs that scene's `ActorSystem` and `PhysicsSystem`; `unloadScene`/loading a different scene tears them down unconditionally, before you can forget to. There's no code path where you construct those systems by hand unless you explicitly opt out with `game.loadScene(def, { manageLifecycle: false })` — a real escape hatch for advanced cases (e.g. sharing one physics world across a seamless open-world boundary), never the default.

The fixed per-frame order, same every frame, no exceptions for node type or tree depth:

1. Input snapshot (frozen for the whole frame — see Input below)
2. Actor mailbox flush, then actor `update()`
3–4. Physics step + collision/sensor dispatch
5. `onUpdate(dt)`
6. Camera/viewport resolve
7. Render

**Overlays** stack an independently-lifecycled scene on top of the main one — HUD, pause menu, minimap — and survive the main scene reloading underneath them:

```typescript
await game.loadOverlay(HudScene)          // no PhysicsSystem unless you pass { physics: {...} }
await game.unloadOverlay()                // pops the most recently loaded overlay
```

`Game.services` is a typed, explicit registry for process-global state (a `ScoreService`, an analytics wrapper, anything that would've been a Godot autoload) — register once, get anywhere, with real types, no ambient globals:

```typescript
class ScoreService { score = 0; add(n: number) { this.score += n } }
game.services.register(ScoreService)
game.services.get(ScoreService).add(10)
```

`game.input` and `game.audio` are the exceptions to "scene-scoped" — they're `Game`-owned singletons that live for the whole process, because held-down keys and playing music don't have anything to do with which scene happens to be loaded right now.

---

## Prefabs and pooling

A prefab is a named template — components plus optionally other prefabs — spawned onto one entity as a unit. There's no live nested-scene tree to override-resolve; `extends` just flattens everything onto that one entity at spawn time.

```typescript
import { definePrefab } from '@emptysock/engine'

const Physical = definePrefab('Physical', [{ def: Transform }, { def: PhysicsBody }])
const Enemy = definePrefab('Enemy', [{ def: Health, overrides: { max: 50 } }], { extends: [Physical] })

scene.spawn(Enemy, { x: 100 })                       // Transform + PhysicsBody + Health, one entity
scene.spawn(Enemy, { x: 100 }, { pool: true })        // pooled — scene.destroy() recycles it instead
```

Pooling folds straight into spawn/destroy — there's no separate `ObjectPool` class to learn. `scene.destroy(pooledEntity)` strips its components and parks it for reuse by the same prefab; game code never branches on whether an entity was pooled. One real wrinkle worth knowing: a pooled-and-destroyed entity's `isAlive` reads `true`, not `false` — the id is deliberately held onto for that prefab's own pool rather than released back for reuse elsewhere. Don't check `isAlive` to ask "was this destroyed" for a pooled entity; check `.has()`/`.get()` on the components you actually care about instead.

See `skills/32-prefabs-pooling.md` for prop-override matching rules and the toolchain's `.d.ts` codegen for JSON-authored prefabs.

---

## Physics

Rapier2D/3D underneath, plain-language properties on top — `velocity`, `type: 'dynamic' | 'static' | 'kinematic'`, `shape` — you never touch a Rapier handle directly.

```typescript
player.add(PhysicsBody, { type: 'dynamic', shape: 'capsule' })
```

Collision/sensor callbacks are a property assignment, not a separate registration call — assigning it *is* registering it:

```typescript
const body = player.get(PhysicsBody)
body.onCollisionEnter = (other, contact) => { if (contact.impactForce > 50) console.log('ouch') }
body.onSensorEnter = (other) => { /* trigger volume entered */ }
```

Two bodies collide unless you tell them not to — collision groups are opt-in tuning, not a prerequisite for anything colliding at all.

3D physics (`PhysicsSystem3D`) needs `await physics.init(...)` and, on scene unload, `physics.destroy()` — the engine calls this for you as part of normal scene teardown when you're not managing lifecycle yourself. Skipping it when you *are* managing it manually leaks WASM linear memory the JS garbage collector can't see, which shows up as an out-of-memory crash on long play sessions with frequent scene changes, not immediately.

---

## Rendering

Attaching `Transform` + a sprite component is the entire contract for "this shows up on screen." No manual PixiJS wiring, ever. `RenderPipeline.mountTilemap()` takes anything shaped like a tile layer, so it works with `@emptysock/tilemap`'s `Tilemap` without the core engine depending on that package.

---

## Input

`InputManager`'s snapshot is frozen for the whole frame — `isDown()`, `keyboard`, `gamepad()`, and `touches` all read that one frozen copy, never live device state mid-frame. This means input can't change out from under your `onUpdate` logic partway through, no matter what else is going on that frame. `Game` never calls `input.attach()` itself; your bootstrap code does that once, which is also what keeps a headless `Game` (tests, server-side logic) from ever touching `window`.

---

## Saving

`SaveSystem` is generic — bind it to a scene and an explicit list of "save-aware" component defs, and it serializes/deserializes them with zero per-component code:

```typescript
import { SaveSystem } from '@emptysock/engine'

const save = new SaveSystem(scene, [Transform, Health, Inventory])
await save.save('slot-1')
await save.load('slot-1')            // migrations run automatically per component, see below
```

If you bump a component's shape, bump `defineComponent(name, defaults, { version: 2 })` and register a migration:

```typescript
save.registerMigration('Health', (oldData, oldVersion) => ({ ...oldData, shield: 0 }))
```

No migration registered for a version mismatch means that one component's data is dropped (with a warning) for that one entity — the rest of the save still loads fine. A schema change never corrupts or aborts the whole load.

With no adapter given, `SaveSystem` defaults to an in-memory store — nothing persists across restarts, which is exactly what you want under tests. The IDE preview and the desktop shell each inject their own real storage adapter (IndexedDB, Tauri's fs plugin) — you never write that adapter code yourself in game logic.

---

## Actors (message-passing)

Unchanged from before, still useful for NPC dialogue, quest triggers, and turn-based messaging where two entities shouldn't hold direct references to each other:

```typescript
class EnemyActor extends Actor {
  receive(msg) { if (msg.type === 'TAKE_DAMAGE') { /* ... */ } }
}
```

`ActorSystem` drains every actor's inbox before any actor's `update()` runs for that frame — a message sent inside `receive()` gets processed in the *same* flush pass, not deferred to next frame. If an actor's message loop sends back to itself unconditionally, that's an infinite loop, not a next-frame deferral. One `ActorSystem` per scene, created and destroyed for you by `Game` — never construct your own unless you've opted out of lifecycle management.

---

## The module packages

Four optional packages sit alongside the core engine. None of them are imported by `@emptysock/engine` itself — you reach for them explicitly, and a game that doesn't need one pays nothing for it.

- **`@emptysock/network`** — Colyseus-backed multiplayer. Mark fields with `networked(componentDef, ['x', 'y'])`, sync with `NetworkSystem.sync()` at a low fixed cadence (not every frame). See `skills/35-network-package.md`.
- **`@emptysock/vn`** — `VNSystem`, the Story Graph runtime for branching dialogue. See `skills/08-story-graph.md`.
- **`@emptysock/battle`** — turn-based `BattleSystem`. See `skills/16-battle-system.md`.
- **`@emptysock/tilemap`** — `TilemapSystem` and `NavMeshSystem`. See `skills/02-navmesh.md` and `skills/17-auto-tile.md`.

Full breakdown of each, plus the trigger for when to reach for it, in the matching skill file above.

---

## Visual scripting compiles to the same API you'd hand-write

A node graph made in the Visual Script Editor compiles down to literal calls against the same public API a code-first dev would write — `scene.spawn(...)`, `variables.setVar(...)`, and so on — not a call into some separate no-code-only runtime. Popping open the generated code from a graph reads like ordinary engine code, because it is. See `skills/29-visual-script-component.md`.

---

## Performance quick rules

- Keep draw calls under **50 per frame** when targeting Raspberry Pi 4 or older mobile.
- Use texture atlases.
- Pool anything that spawns frequently — see Prefabs and pooling above.
- Avoid allocating inside `onUpdate`.
- Prefer `scene.each(...)` over per-entity `.get()` once you're touching more than a handful of entities a frame.

---

## Further reference

- `ai/api-reference.json` — full machine-readable API, in this repo.
- `skills/` — one topic-scoped guide per system, in this repo.
- Engine documentation (lives in the engine repo): `docs/getting-started/`, `docs/guides/`, `docs/reference/`, `docs/tutorials/`.
