# ECS core — Entity, Scene, defineComponent, Game

**Use this when** you're spawning entities, defining components, wiring a scene's `onLoad`/`onUpdate`/`onUnload`, or trying to understand why `entity.get()` returns `undefined`. This is the foundation every other skill file assumes you already know.

---

## The shape of it

An entity is a lightweight, cheap-to-copy handle — not a class you extend, not a bag of state. The state lives in component storage; `Entity` is just your way of pointing at it.

```typescript
import { defineComponent, Scene } from '@emptysock/engine'

const Health = defineComponent('Health', () => ({ current: 100, max: 100 }))

const scene = new Scene()
const goblin = scene.spawn('Goblin')
goblin.add(Health, { current: 30 })

goblin.has(Health)          // true
const health = goblin.get(Health)   // proxy onto the real data, or undefined
if (health !== undefined) health.current -= 10
scene.destroy(goblin)
```

`defineComponent(name, createDefaults, options?)`:

- `name` is the real identity — it's a string, not the object `defineComponent` returns, because hot-reloading the file that defines a component re-evaluates the module and hands you a fresh object reference every time. The engine treats two defs with the same name as "the same" component, so pick a name once and don't rename it casually.
- `createDefaults` produces a fresh defaults object per new component instance.
- `options.version` (defaults to `1`) matters for `SaveSystem` — see `skills/33-save-system.md`.
- `options.schema` is an optional per-field description the IDE's Inspector uses for a typed editor instead of a raw JSON box. Entirely optional; nothing breaks without it.

Every field on a component has to be `Serializable` — strings, numbers, booleans, `null`, arrays, and plain objects made of the same. No functions, no class instances. This is the deal that makes `SaveSystem` work with zero per-component save code. If you genuinely need a callback-shaped property on a component (collision handlers are the standing example — see `PhysicsBody` in `skills/00-quickstart.md`), it lives in a small side-table behind the component, not as a field — you still assign it like a property, it's just not stored as component data underneath.

## `.get()` vs `.each()`

`entity.get(Component)` is what you reach for touching one entity — an AI script, a player controller, anything per-entity. `scene.each(...)` is the bulk path:

```typescript
scene.each(Transform, PhysicsBody, (transform, body, entity) => {
  transform.x += body.velocity.x * dt
})
```

Same components, same object — no separate import, no "advanced" module boundary. `each` reads straight off the raw arrays and skips the proxy allocation `.get()` does per entity, which is the difference that shows up once you're touching hundreds of entities a frame and doesn't matter at all for a single boss's `onUpdate`. Think `Array.forEach`, not "a scary database query" — that's deliberately the whole mental model.

## Scenes don't own physics or actors — `Game` does

A `Scene` is just the ECS world. Physics, actors, and rendering are wired to a scene's lifecycle by `Game`, and `Game` owns creating and destroying them:

```typescript
import { Game, defineScene } from '@emptysock/engine'

const GameScene = defineScene({
  onLoad(scene, { actors, physics, input, audio }) {
    // actors and physics already exist — never `new ActorSystem()` yourself here.
  },
  onUpdate(dt) {
    // runs every frame, after physics/actors/collision, before render
  },
  onUnload(scene) {
    // called before actors/physics get torn down
  },
})

const game = new Game()
await game.loadScene(GameScene)
```

Loading a new scene (or calling `unloadScene`) tears down the previous scene's `ActorSystem`/`PhysicsSystem` unconditionally, before you get the chance to forget to. The one escape hatch is `game.loadScene(def, { manageLifecycle: false })`, which hands you the raw systems to own yourself — for something like sharing one physics world across a seamless open-world boundary. It's opt-in per call, not a global switch, and you shouldn't reach for it unless you have a specific reason.

`onUpdate` cannot be async. In TypeScript this is a real compile error (the type is `(dt: number) => void`), because the game loop calls it synchronously and never awaits it — any work scheduled after an `await` inside runs at a time the frame budget has no idea about, and errors thrown after that point never get caught. Use `entity.startCoroutine(...)` for multi-frame work. In plain JavaScript you lose the compile-time catch but the engine still warns loudly at runtime ("`onUpdate` returned a Promise. That's not a thing here — use `entity.startCoroutine()` instead.") if it detects a Promise coming back.

## Overlays: a second, independent scene on top

```typescript
await game.loadOverlay(HudScene)
await game.unloadOverlay()       // pops the most recently loaded overlay by default
```

Overlays are how you do a HUD, pause menu, or minimap without coupling that UI's lifecycle to the main scene's — an overlay survives the main scene reloading underneath it, and you can stack more than one. They get their own `ActorSystem`, same one-per-scene guarantee as the main scene, but no `PhysicsSystem` unless you explicitly pass `{ physics: {...} }` — a HUD doesn't need to pay for a physics world it'll never use.

## Services: the typed alternative to a global singleton

```typescript
class ScoreService {
  score = 0
  add(n: number) { this.score += n }
}

game.services.register(ScoreService)
game.services.get(ScoreService).add(10)     // typed, no cast needed
```

One `ServiceRegistry` per `Game`, alive for the whole process, never reset by scene loads. This is the explicit, typed answer to "I need some shared state everywhere" — save data, audio mixing state, third-party plugin wrappers, all fit here as one mechanism instead of a pile of ad hoc singletons.

## Input and audio are `Game`-owned, not scene-owned

`game.input` and `game.audio` persist for the life of the `Game` instance and are never recreated on scene load — held-down keys and currently-playing music have nothing to do with which scene is loaded right now. If you want scene-scoped sound (stop this scene's music on unload), do it explicitly from that scene's own `onUnload`; the engine doesn't try to guess which sounds "belong" to which scene.
