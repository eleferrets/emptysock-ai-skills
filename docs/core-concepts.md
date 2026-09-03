# Core Concepts

The ideas behind EmptySock. Read this once, then the skill files make immediate sense.

Don't worry if some of this feels abstract at first — come back to it after you've made something, and it'll click.

---

## Entity-Component System (ECS)

EmptySock uses ECS — the same architecture as Unity, Godot, and Bevy.

**The idea in plain English:** instead of building a big inheritance tree (`Enemy extends Character extends GameObject`), you build game objects by snapping small, reusable pieces together.

- **Entity** — a named container with a position. Has no logic on its own. Think of it like an empty box with a label.
- **Component** — a behaviour or data bundle you attach to an entity (`Sprite`, `PhysicsBody`, `Animator`). Think of it like a LEGO brick.
- **System** — engine code that runs every frame, looking for entities that have certain components and doing something with them.

**Why this matters:**

```typescript
// The old way (inheritance): Player, Enemy, and Projectile all need physics,
// but they don't all share the same base class, so you end up copying code.

// The ECS way: any entity can have any component.
const player = this.createEntity('Player')
player.addComponent(Sprite, { texture: 'hero.png' })
player.addComponent(PhysicsBody, { bodyType: 'dynamic' })
player.addComponent(Animator, { spritesheet: 'hero.esanim' })

const bullet = this.createEntity('Bullet')
bullet.addComponent(Sprite, { texture: 'bullet.png' })
bullet.addComponent(PhysicsBody, { bodyType: 'dynamic', ccd: true })
// Same PhysicsBody component — the same physics system handles both automatically.
```

You build things by combining, not by inheriting. It's like building with LEGO instead of carving from a single block of wood.

One important rule: inside a Scene class method, create entities with `this.createEntity('Name')` — not `scene.createEntity()`. The `this` refers to the scene you're inside.

---

## Scene Graph

All entities exist inside a **scene**. Scenes are:

- Loaded one at a time (or stacked via push/pop for overlays like pause screens).
- Self-contained: entities, physics world, lighting, and camera all belong to a scene.
- Defined as TypeScript classes that extend `Scene`.

Entities can be **parented** to other entities. A child entity's position, rotation, and scale are relative to its parent. Destroying a parent destroys all children.

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
const t = entity.requireComponent(Transform)
t.x += speed * dt   // 'speed' is pixels per second — consistent on any device
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

One important note: if you use `PhysicsSystem3D`, always call `physics.destroy()` when your scene unloads. The physics engine allocates memory outside of JavaScript's reach, and if you don't free it, the memory never gets released. Call it in `onDestroy`.

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
import { Input } from '@emptysock/engine'

Input.isDown('ArrowRight')          // keyboard right arrow
Input.isDown('gamepad0/DPadRight')  // gamepad D-pad right
Input.axis('Horizontal')            // keyboard -1/0/1 or gamepad stick -1..1
Input.pointer.position              // mouse on desktop, first touch on mobile
```

Import `Input` at the top of the file and call it directly — it's a static class, so you don't need to create an instance.

---

## Saves

Save data is stored per-slot. Each slot is a named JSON blob plus metadata. Always validate save data through a Zod schema when loading — saves can be from older versions of your game and the shape may have changed since.

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
  SceneManager.transition('WinScene', { effect: 'iris' })
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

Not upfront. If you can write JavaScript, you can copy the patterns from this guide and the skills files and be productive immediately. TypeScript is JavaScript with optional labels — you'll pick it up gradually as you make things.

**Can I use JavaScript instead?**

Technically yes — TypeScript files are just JavaScript with type annotations you can leave out. But the engine's autocomplete, error checking, and all the examples here assume TypeScript. Leaving out types means losing the error-catching benefits, and you'll see warnings in the editor. Start with TypeScript — it's easier than it looks.

**What if I get a red error?**

Red underlines in the editor mean TypeScript found a type mismatch or a possibly-null value. Hover over the underline to read the message. Common causes:

- You forgot to check for `null` before using a value that might be null.
- You passed a `string` where a `number` was expected (or vice versa).
- You misspelled a method name.

Red errors in the console after hitting Play are runtime errors — something went wrong while the code was actually running. Read the message and the line number; they're usually pretty descriptive.

**How do I make something appear on screen?**

The quickest way is to create a canvas in `onLoad` and draw to it with the 2D canvas API — no assets required. See the example in `docs/getting-started.md`. Once you're comfortable, move to using `Sprite` components with texture files for anything you want to stay in your game long-term.
