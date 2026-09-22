# Core Concepts

The ideas behind EmptySock. Read this once, then the skill files make immediate sense.

Don't worry if some of this feels abstract at first — come back to it after you've made something, and it'll click.

---

## Entity-Component System (ECS)

EmptySock uses ECS — the same underlying idea as Unity, Godot, and Bevy, though the actual object you hold is closer to Bevy's than Unity's.

**The idea in plain English:** instead of building a big inheritance tree (`Enemy extends Character extends GameObject`), you build game objects by snapping small, reusable pieces of data together.

- **Entity** — a lightweight handle, not a container holding state itself. Think of it as a claim ticket, not the coat.
- **Component** — a plain data bundle you attach to an entity (`Sprite`, `PhysicsBody`, an `Health` component you define yourself). Think LEGO brick.
- **Scene** — owns the entities and their components for one loaded level or screen. `Game` is what wires physics, actors, and rendering to a scene's lifecycle.

**Why this matters:**

```typescript
// The old way (inheritance): Player, Enemy, and Projectile all need physics,
// but they don't all share the same base class, so you end up copying code.

// The ECS way: any entity can have any component.
const player = scene.spawn('Player')
player.add(Sprite, { texture: 'hero.png' })
player.add(PhysicsBody, { type: 'dynamic' })

const bullet = scene.spawn('Bullet')
bullet.add(Sprite, { texture: 'bullet.png' })
bullet.add(PhysicsBody, { type: 'dynamic', ccd: true })
// Same PhysicsBody component — the same physics system handles both automatically.
```

You build things by combining, not by inheriting. It's like building with LEGO instead of carving from a single block of wood. `entity.get(Sprite)` hands you back a live view onto that component's data — write to it and the write lands straight in the engine's storage, no separate "commit" step.

Define your own components with `defineComponent`:

```typescript
import { defineComponent } from '@emptysock/engine'

const Health = defineComponent('Health', () => ({ current: 100, max: 100 }))
player.add(Health, { current: 80 })
```

See `skills/32-ecs-core.md` for the full rules on this (there's a real reason component fields have to stay plain data — it's what lets `SaveSystem` save any component with zero extra code, see `skills/33-save-system.md`).

---

## Scenes

All entities exist inside a **scene**. Scenes are:

- Loaded one at a time via `Game.loadScene()` — or stacked as independent overlays via `Game.loadOverlay()` for HUDs and pause screens.
- Self-contained: their entities, physics world, and actor system belong to that one scene, created and destroyed for you automatically by `Game`.
- Defined with `defineScene({ onLoad, onUpdate, onUnload })`, not by subclassing.

There's no live parent/child entity tree — if you want a template made of several components spawned together, that's a **prefab** (`definePrefab`), flattened onto one entity at spawn time. See `skills/34-prefabs-pooling.md`.

---

## The Game Loop

EmptySock runs a game loop (default 60 times per second, called 60 FPS — frames per second). Each frame:

1. Process input events
2. Call `onUpdate(dt)` on your scene
3. Step the physics simulation
4. Render everything to the screen

`dt` in `onUpdate` is the real elapsed time since the last frame, in seconds. At 60 FPS this is roughly 0.016. You use it to make movement **frame-rate independent** — so the game feels the same whether it's running at 30fps or 120fps.

```typescript
// Without dt: moves a fixed number of pixels per frame
// → runs twice as fast on a 120fps screen, half speed on a 30fps screen
t.x += 5

// With dt: moves a fixed number of pixels per *second*, regardless of frame rate
const t = entity.get(Transform)
if (t !== undefined) t.x += speed * dt   // 'speed' is pixels per second — consistent on any device
```

Always use `dt` for anything that moves or changes over time.

---

## Delta Time and Game Speed

The engine runs at `gameSpeed` FPS (default 60). You can slow time down globally — useful for slow-motion effects:

```typescript
Engine.timeScale = 0.5   // half speed — everything slows down
Engine.timeScale = 2.0   // double speed
Engine.timeScale = 1.0   // normal
```

`dt` passed to `onUpdate` reflects the time scale — so your movement code doesn't need to know about it. Coroutine `waitSeconds()` also respects time scale. `Timer.after()` runs in real time and doesn't slow down.

---

## Physics

EmptySock uses a Rust physics library compiled to WebAssembly. The physics world runs alongside the game loop. Every entity with a `PhysicsBody` component participates in the simulation.

- **Fixed** bodies never move. Use for walls, floors, platforms.
- **Dynamic** bodies respond to gravity and forces. Use for enemies, crates, coins.
- **Kinematic** bodies are moved by code, but they push dynamic bodies. Use for moving platforms and character controllers.

`CharacterController` is a pre-built kinematic controller that handles slope climbing, step snapping (stairs), and ground detection. Use it for players. Use raw `PhysicsBody` for everything else.

Collision and sensor callbacks are a direct property assignment on the `PhysicsBody` you already hold — `body.onCollisionEnter = (other, contact) => {...}` — assigning the property is registering it.

One important note: if you're using `PhysicsSystem3D` with manual lifecycle management, always call `physics.destroy()` when your scene unloads. The physics engine allocates memory outside of JavaScript's reach, and if you don't free it, the memory never gets released. In the normal case (you didn't pass `{ manageLifecycle: false }` to `loadScene`), `Game` calls this for you automatically on scene teardown — you only need to remember it yourself if you opted out of automatic lifecycle management.

---

## Rendering

The renderer uses WebGL2 (or WebGPU where available).

Key ideas:

**Draw calls:** each texture switch is a new draw call. Fewer draw calls = faster rendering. Group sprites that share a texture. The IDE auto-packs texture atlases at export time, so you usually don't need to think about this.

**Z-order:** entities render back-to-front. Control the order with `entity.zIndex` — higher = in front.

**2D lighting:** optional. Enable with `lighting: true` in your scene config. Add `PointLight`, `DirectionalLight`, or `SpotLight` components to entities that should cast light.

---

## Audio

Audio is loaded from `assets/audio/` and played by name. Audio is grouped:

- `music` — background music, usually looping
- `sfx` — sound effects
- `ui` — button clicks, UI sounds
- `voice` — dialogue voice lines

Each group has its own volume control. Players expect to be able to turn music down but keep sound effects loud, so keeping them separate is worth it from day one.

Spatial audio pans and attenuates sounds based on distance from the camera (listener position).

---

## Input

Input is unified across keyboard, mouse, touch, and gamepad. The same code works on desktop and mobile:

```typescript
import { InputSystem } from '@emptysock/engine'

// In onLoad:
const input = new InputSystem()
input.attach()   // register event listeners

// In onUpdate — call flush() first, then read state:
input.flush()
input.isKeyDown('ArrowRight')   // keyboard right arrow — held every frame
input.isKeyPressed('Space')     // true only on the frame the key went down
input.mouseX / input.mouseY     // mouse position on desktop, first touch on mobile
```

Create one `InputSystem` instance in `onLoad`, call `attach()` to register listeners, `flush()` at the top of every `onUpdate`, and `detach()` in `onDestroy`.

---

## Saves

`SaveSystem` saves a scene's components generically — bind it once to the components you want persisted (`new SaveSystem(scene, [Transform, Health])`) and it handles the serialization itself, no per-component code needed. If a component's shape changes later, bump its version and register a migration rather than hoping old saves happen to still fit — see `skills/33-save-system.md`.

---

## Timers

Use `Timer` to run code after a delay or on a repeating interval. Never use `setTimeout` in game logic — it ignores the engine's time scale and doesn't get cleaned up automatically with your scene.

```typescript
import { Timer } from '@emptysock/engine'

// Run once after 2 seconds
const handle = Timer.after(2, () => {
  console.log('2 seconds passed!')
})

// Run every 1 second
const repeatingHandle = Timer.every(1, () => {
  spawnEnemy()
})

// Cancel a timer early if you need to
handle.cancel()
```

`Timer.after` and `Timer.every` both return a `TimerHandle`. Keep the handle if you might need to cancel the timer — for example, in `onDestroy` so a timer doesn't fire after the scene is gone.

---

## Coroutines

Coroutines are generator functions that run over multiple frames. They're the replacement for `setTimeout`, `async/await`, or frame counters when you want to sequence events over time.

```typescript
entity.startCoroutine(function* () {
  yield waitSeconds(2.0)             // pause for 2 seconds
  spawnBoss()
  yield waitForEvent('boss_dead')    // pause until a custom event fires
  SceneManager.transition('WinScene', { effect: 'wipe' })
})
```

Coroutines respect `Engine.timeScale` — they slow down in slow motion just like everything else. They're deterministic and easy to read.

---

## TypeScript

EmptySock is written in TypeScript and the engine API is fully typed. That might sound scary if you're new, but it's genuinely helpful — the editor tells you when you've made a mistake before you even run the game.

Here are the rules the engine follows, and that your game code should follow too:

**No `!` non-null assertions.** This shortcut hides bugs instead of fixing them. Use `?.` optional chaining or a null check instead:

```typescript
// Bad — crashes silently if _canvas is null
this._canvas!.getContext('2d')

// Good — does nothing if _canvas is null
this._canvas?.getContext('2d')

// Also good — explicit check you can add a fallback to
const canvas = this._canvas
if (canvas === null) return
canvas.getContext('2d')
```

**No `any`.** Using `any` turns off type checking. Use `unknown` with a type guard instead, or use the actual type.

**Validate external data.** Save files, JSON assets, and network responses can be anything — always run them through a schema (Zod) before trusting their shape.

These rules exist for a practical reason: games run on hardware you can't fix remotely. A type error caught at compile time is a crash you avoid on a player's device.

---

## Common questions

**Do I need to learn TypeScript?**

No, genuinely. JavaScript is a fully first-class citizen here, not a stripped-down fallback — both languages run against the exact same API, same objects, same methods. The one thing TypeScript buys you that JavaScript can't get at compile time is catching an accidentally-`async onUpdate` before you even run the game; in JavaScript the engine catches the same mistake at runtime instead, with a console warning. Everything else works identically either way.

**Can I use JavaScript instead?**

Yes, without caveats. You'll lose some autocomplete convenience compared to TypeScript, but no functionality and no supported patterns.

**What if I get a red error?**

Red underlines in the editor mean TypeScript found a type mismatch or a possibly-null value. Hover over the underline to read the message. Common causes:

- You forgot to check for `null` before using a value that might be null.
- You passed a `string` where a `number` was expected (or vice versa).
- You misspelled a method name.

Red errors in the console after hitting Play are runtime errors — something went wrong while the code was actually running. Read the message and the line number; they're usually pretty descriptive.

**How do I make something appear on screen?**

The quickest way is to create a canvas in `onLoad` and draw to it with the 2D canvas API — no assets required. See the example in `docs/getting-started.md`. Once you're comfortable, move to using `Sprite` components with texture files for anything you want to stay in your game long-term.

---

## Further reading

The companion skill files in `skills/` cover each system in depth. For the engine's own
full documentation, see the engine repository. It uses a Unity/Unreal-style layout:

| Section | Path in engine repo |
|---|---|
| Install and first project | `docs/getting-started/` |
| How-to guides (physics, NavMesh, exports…) | `docs/guides/` |
| Full API reference | `docs/reference/` |
| Step-by-step tutorials | `docs/tutorials/` |
