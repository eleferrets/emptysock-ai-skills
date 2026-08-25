# Core Concepts

The ideas behind EmptySock. Read this once, then the skill files make immediate sense.

---

## Entity-Component System (ECS)

EmptySock uses ECS — the same architecture as Unity, Godot, and Bevy.

**The idea:** instead of inheritance hierarchies (`Enemy extends Character extends GameObject`),
you build game objects by combining small, reusable pieces.

- **Entity** — a named container with a position. Has no logic on its own.
- **Component** — a data and behaviour bundle attached to an entity (`Sprite`, `PhysicsBody`, `Animator`).
- **System** — engine code that runs every frame, operating on entities that have certain components.

**Why this is better:**

```typescript
// Old way (inheritance): Player, Enemy, and Projectile all need physics,
// but only some of them share a base class. You end up with duplication.

// ECS way: any entity can have any component.
const player = scene.createEntity('Player')
player.addComponent(Sprite, { texture: 'hero.png' })
player.addComponent(PhysicsBody, { bodyType: 'dynamic' })
player.addComponent(Animator, { spritesheet: 'hero.esanim' })

const bullet = scene.createEntity('Bullet')
bullet.addComponent(Sprite, { texture: 'bullet.png' })
bullet.addComponent(PhysicsBody, { bodyType: 'dynamic', ccd: true })
// Same PhysicsBody component, same physics system handles both.
```

You build things by composing, not inheriting.

---

## Scene Graph

All entities exist inside a **scene**. Scenes are:

- Loaded one at a time (or stacked via push/pop for overlays like pause screens).
- Self-contained: entities, physics world, lighting, camera all belong to a scene.
- Defined as TypeScript classes that extend `Scene`.

Entities can be **parented** to other entities. A child entity's position, rotation, and scale
are relative to its parent. Destroying a parent destroys all children.

---

## The Game Loop

EmptySock runs a fixed-FPS game loop (default 60 FPS). Each frame:

1. Process input events
2. Call `onUpdate(dt)` on all scene components
3. Step the physics simulation
4. Call `onFixedUpdate(dt)` (fixed 1/60s timestep, good for physics-coupled logic)
5. Render the scene (PixiJS → WebGL/WebGPU)
6. Present to screen (vsync-locked via `requestAnimationFrame`)

`dt` in `onUpdate` is the real elapsed time since the last frame, in seconds.
At 60 FPS this is ~0.0167. Use it to make movement frame-rate independent:

```typescript
entity.translate(speed * dt, 0)  // moves 'speed' pixels per second regardless of FPS
```

---

## Delta Time and Game Speed

The engine runs at `gameSpeed` FPS (default 60). You can change this per-scene or
slow time down globally for slow-motion effects:

```typescript
Engine.timeScale = 0.5   // half speed — everything slows down
Engine.timeScale = 2.0   // double speed
Engine.timeScale = 1.0   // normal
```

`dt` passed to `onUpdate` reflects the time scale. Coroutine `waitSeconds()` also respects
time scale. Timer `Timer.after()` does not — timers run in real time.

---

## Physics

EmptySock uses Rapier2D (Rust → WASM). The physics world runs alongside the game loop.

Every entity with a `PhysicsBody` component participates in the simulation.

- **Fixed** bodies never move. Use for walls, floors, platforms.
- **Dynamic** bodies respond to gravity and forces. Use for enemies, crates.
- **Kinematic** bodies are moved by code, but push dynamic bodies. Use for platforms, character controllers.

`CharacterController` is a pre-built kinematic controller that handles:
- Slope climbing and descent
- Step snapping (stairs)
- Ground detection

Use `CharacterController` for players. Use raw `PhysicsBody` for everything else.

---

## Rendering

The renderer is PixiJS v8 (WebGL2, with WebGPU where available).

Key ideas:

**Draw calls:** each texture switch is a new draw call. Fewer draw calls = faster rendering.
Group sprites that share a texture. The IDE auto-packs texture atlases at export time.

**Z-order:** entities render back-to-front. Control Z with `entity.zIndex` (higher = in front).

**2D lighting:** optional. Enable with `lighting: true` in your scene config.
Each `PointLight`, `DirectionalLight`, or `SpotLight` component casts light.
Sprites can have a `normalMap` texture for 3D-looking depth under 2D lighting.

---

## Audio

Audio is loaded from `assets/audio/` and played by name. Audio is grouped:

- `music` — background music, usually looping
- `sfx` — sound effects
- `ui` — button clicks, UI sounds
- `voice` — dialogue voice lines

Each group has a master volume control. Players typically expect to control these separately.

Spatial audio pans and attenuates sounds based on distance from the camera (listener position).

---

## Input

Input is normalised across keyboard, mouse, touch, and gamepad. The same code works on
desktop and mobile:

```typescript
Input.isDown('ArrowRight')          // keyboard right arrow
Input.isDown('gamepad0/DPadRight')  // gamepad D-pad right
Input.axis('Horizontal')            // keyboard -1/0/1 or gamepad stick -1..1
Input.pointer.position              // mouse on desktop, first touch on mobile
```

---

## Saves

Save data is stored per-slot. Each slot is a named JSON blob plus metadata.
Always validate save data through a Zod schema when loading — saves can be from
older versions of the game and the schema may have changed.

---

## Coroutines

Coroutines are generator functions that run over multiple frames.
They're the replacement for `setTimeout`, `async/await`, or frame counters
when you want to sequence events over time in game logic.

```typescript
entity.startCoroutine(function* () {
  yield waitSeconds(2.0)    // pause for 2 seconds
  spawnBoss()
  yield waitForEvent('boss_dead')
  SceneManager.transition('WinScene', { effect: 'iris' })
})
```

Coroutines respect `Engine.timeScale`. They're deterministic and easy to reason about.

---

## TypeScript

EmptySock is TypeScript-strict. Every property is typed. The API is designed so that
correct code is easy to write and incorrect code fails at compile time.

Rules:
- Zero `any`. Use `unknown` with Zod or explicit type guards.
- Zero `!` non-null assertions. Use `?.` and fallbacks.
- All external data (saves, JSON files) validated through Zod before use.

These rules exist because games run on hardware you cannot fix remotely.
A type error you catch at compile time is a crash you avoid on a player's Raspberry Pi.
