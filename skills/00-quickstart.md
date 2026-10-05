# EmptySock — Quickstart

Everything you need for a typical game task, in one page. For the full story on the entity/component/scene core, see `skills/32-ecs-core.md`.

---

## Engine boot and scene skeleton

```typescript
import { Game, defineScene } from '@emptysock/engine'

const GameScene = defineScene({
  async onLoad(scene, ctx) {
    // ctx: { scene, actors, physics, input, audio, variables, globals, signals,
    //        fonts, assets, plugins, localisation, viewport, window, restoreCarried(), restoreRoom() }
    // Game created actors/physics for you; never `new ActorSystem()` yourself here.
  },
  onUpdate(dt) {
    // runs every frame, after physics/actor/collision, before render.
    // no async here: it's a type error. Use entity.startCoroutine() instead.
  },
  onUnload(scene) {
    // called before this scene's actors/physics are torn down (automatically)
  },
})

const game = new Game()
await game.loadScene(GameScene)
// each frame, from your host's render loop: game.update(dt)
```

`game.loadOverlay(HudScene)` stacks an independent scene on top (HUD, pause menu) that survives a main-scene reload underneath it. `game.services` is a typed registry for process-global state; see `skills/32-ecs-core.md`. Scenes are plain objects from `defineScene`, not classes; keep scene state in module variables or closures.

---

## Entities and components

Entities are handles, not classes to subclass. Components are plain data from `defineComponent`.

```typescript
import { defineComponent, Transform, Sprite, PhysicsBody, waitSeconds } from '@emptysock/engine'

const player = scene.spawn('Player')
player.add(Transform, { x: 0, y: 0 })
player.add(Sprite, { texturePath: 'hero.png', anchorX: 0.5, anchorY: 1.0 })
player.add(PhysicsBody, { shape: 'capsule', type: 'dynamic' })

player.has(PhysicsBody)              // boolean
const body = player.get(PhysicsBody) // T | undefined: proxy onto live data

// Coroutines on the entity: the escape hatch for multi-frame work
player.startCoroutine(function* () {
  yield waitSeconds(1.0)
})

// Destroy: same call whether or not the entity was spawned pooled
scene.destroy(player)
```

Define your own components with `defineComponent(name, () => defaults, options?)`; see `skills/32-ecs-core.md` for the rules (serializable fields only, name is the real identity across hot-reload). For touching many entities at once, use `scene.each(Transform, PhysicsBody, (t, b, entity) => {...})`.

---

## Rendering (RenderPipeline)

Attaching `Transform` + `Sprite` to an entity is the entire contract for "this shows up on screen". See `skills/23-rendering.md` for tilemaps, layers, and custom texture loading.

```typescript
import { Game, RenderPipeline } from '@emptysock/engine'

const render = new RenderPipeline()
await render.init({ width: 1280, height: 720 })
document.body.appendChild(render.canvas)

game.attachRenderer(render)   // Game then calls render.renderFrame(main, overlays) for you each update
// when shutting down: render.destroy()
```

---

## 2D physics

`PhysicsBody` is a component backed by Rapier2D; `ctx.physics` (a `PhysicsSystem`) is created and stepped by `Game`.

```typescript
import { PhysicsBody, getPhysicsBody } from '@emptysock/engine'

player.add(PhysicsBody, { shape: 'capsule', type: 'dynamic', width: 24, height: 48 })

// In onUpdate: drive the body by writing a new velocity object
const body = player.get(PhysicsBody)
if (body !== undefined) {
  const left = input.isDown('left')
  const right = input.isDown('right')
  const h = right ? 1 : left ? -1 : 0
  body.velocity = { x: h * 200, y: body.velocity.y }
}

// Collision/sensor callbacks are assigned through getPhysicsBody(); assigning IS registering
const handle = getPhysicsBody(player)
if (handle !== undefined) {
  handle.onCollisionEnter = (other, contact) => {
    if (contact.impactForce > 50) console.log('Ouch.')
  }
  handle.onSensorEnter = (other) => { /* trigger volume entered */ }
}
```

`PhysicsBody` fields: `type` ('dynamic' | 'static' | 'kinematic'), `shape` ('box' | 'circle' | 'capsule'), `width`, `height`, `radius`, `density`, `friction`, `restitution`, `isSensor`, `position`, `rotation`, `velocity`. Callbacks: `onCollisionEnter`, `onCollisionExit`, `onSensorEnter`, `onSensorExit`, `onSensorStay`. `PhysicsSystem` also offers `raycast(...)`, `overlapCircle(center, radius)` and `getBodyState(entity)`. Bodies collide with everything by default. There is no built-in character controller; compose movement from velocity and your own ground checks.

---

## 3D physics (Rapier3D, must await init)

```typescript
import { PhysicsSystem3D } from '@emptysock/engine'

// In onLoad:
const physics = new PhysicsSystem3D()
await physics.init({ gravity: { x: 0, y: -9.81, z: 0 } })

const box = physics.addBody({
  bodyType: 'dynamic',
  shape: 'box',
  halfExtents: { x: 0.5, y: 0.5, z: 0.5 },
  position: { x: 0, y: 5, z: 0 },
})

// In onUpdate:
physics.update(dt)
const pos = box.getPosition() // { x, y, z }

// In onUnload, REQUIRED, frees WASM memory:
physics.destroy()
```

---

## Actor Model (message-driven logic)

```typescript
import { Actor, ActorSystem, type Message } from '@emptysock/engine'

type TakeDamageMsg = { type: 'TAKE_DAMAGE'; amount: number }

class EnemyActor extends Actor {
  receive(msg: Message): void {
    const m = msg as TakeDamageMsg
    if (m.type === 'TAKE_DAMAGE') { /* handle m.amount */ }
  }
  update(dt: number): void { /* per-frame AI */ }
}

const system = new ActorSystem()
system.register(new EnemyActor('enemy-1'))
system.send('enemy-1', { type: 'TAKE_DAMAGE', amount: 25 })
system.update(dt)   // flush mailboxes then run update() on all actors
system.destroy()    // on unload
```

In a normal scene you don't construct `ActorSystem` yourself: `Game.loadScene()` creates and destroys one per scene and hands it to you as `actors`.

---

## Input

Game code reads input through the game-owned `InputManager` (`ctx.input` / `game.input`): named actions mapped to keys and gamepad inputs, read from a per-frame frozen snapshot.

```typescript
import { Game, InputManager } from '@emptysock/engine'

const input = new InputManager({
  jump:  [{ kind: 'key', code: 'Space' }, { kind: 'gamepadButton', index: 0 }],
  left:  [{ kind: 'key', code: 'ArrowLeft' }, { kind: 'gamepadAxis', axis: 0, threshold: -0.5 }],
  right: [{ kind: 'key', code: 'ArrowRight' }, { kind: 'gamepadAxis', axis: 0, threshold: 0.5 }],
})
input.attach()   // host bootstrap only: starts DOM listeners (never called by Game itself)

// In onUpdate (Game calls input.snapshot() once per frame before your code):
if (input.wasPressed('jump')) { /* edge: pressed this frame */ }
if (input.isDown('right'))    { /* held */ }
if (input.wasReleased('jump')) { /* edge: released this frame */ }

// Raw escape hatches
input.keyboard.isDown('KeyW')
input.gamepad(0).axis(0)
input.pointers   // PointerState[]; also input.gestures, input.wheelEvents
```

Touch and pointer details: `skills/05-touch-input.md`, `skills/25-pointer-system.md`. `InputSystem`, `PointerSystem` and `GamepadSystem` are the raw device layers `InputManager` wraps; they are exported from `@emptysock/engine` too, but game code normally goes through `InputManager`. Remapping and persistence: `skills/28-accessibility-debugging.md`.

---

## Particles

`ParticleEmitter` is a plain simulation class (not a component). Mount it on a render layer to draw it.

```typescript
import { ParticleEmitter } from '@emptysock/engine'

const emitter = new ParticleEmitter({
  texture: 'fx/spark.png',                 // optional; default is a white square
  emissionRate: 30,
  lifetime: { min: 0.8, max: 1.5 },
  velocity: { x: { min: -60, max: 60 }, y: { min: -120, max: -60 } },
  maxParticles: 100,
})
emitter.x = 200
emitter.y = 300
await render.mountParticles(emitter, 'default')   // RenderPipeline

emitter.emit(24)    // burst of 24 immediately
emitter.stop()      // stop new particles; existing ones finish
// each frame: emitter.update(dt)
// on unload: render.unmountParticles(emitter)
```

See `skills/09-particles.md`.

---

## Audio

`ctx.audio` / `game.audio` is the game's `AudioSystem` (Howler-backed; the `AudioSystem` class is exported, but use the instance the game already owns). Load sounds by id, then play by id.

```typescript
audio.load('jump_sfx', 'assets/sfx/jump.ogg', { group: 'sfx' })
audio.load('level_theme', 'assets/music/level.ogg', { group: 'music', loop: true })
audio.play('jump_sfx')
audio.play('level_theme')
audio.setGroupVolume('sfx', 0.8)
audio.stop('level_theme')
audio.masterVolume = 0.9
```

Also available: `pause(id)`, `setPitch(id, rate)`, `unload(id)`, `duck(bus, amount, fadeTime?)`, `endDuck(bus)`, `defineSnapshot(name, busVolumes)`, `transitionToSnapshot(name, duration?)`. Call `audio.update(dt)` each frame if you use ducking or snapshots.

---

## Camera

`CameraSystem` is an instance you create, then attach to the PixiJS stage.

```typescript
import { CameraSystem } from '@emptysock/engine'
import type { CameraBounds } from '@emptysock/engine'

const camera = new CameraSystem()
camera.attach(stage)                         // your PixiJS Container (e.g. render.stage)
camera.setViewSize(1280, 720)

camera.setFollow(() => ({ x: playerX, y: playerY }))   // smooth follow
camera.setLerpFactor(0.1)                   // 0 = no movement, 1 = instant

camera.snapTo(0, 0)                         // instant
camera.moveTo(500, 300)                     // smooth to target
camera.zoomTo(2.0)                          // smooth
camera.snapZoom(1.0)                        // instant
camera.shake(6, 0.3)                        // intensity px, duration seconds

const bounds: CameraBounds = { minX: 0, minY: 0, maxX: 4000, maxY: 2000 }
camera.setBounds(bounds)
camera.setBounds(null)                      // remove clamping

const worldPos = camera.screenToWorld(sx, sy)
const screenPos = camera.worldToScreen(wx, wy)

camera.update(dt)                           // every frame
camera.destroy()                            // on unload
```

---

## Gamepad

Gamepads go through the game's `InputManager`. Bind buttons and axes to actions (see Input above), or read raw state from the frozen frame snapshot:

```typescript
const pad = input.gamepad(0)            // GamepadSnapshot for pad index 0
if (pad.connected) {
  const jumpHeld = pad.isButtonDown(0)  // A button
  const leftX = pad.axis(0)             // left stick X, -1..1
}
```

`GamepadSystem` (rumble, raw polling) is exported from `@emptysock/engine`; `InputManager` already wraps one, so prefer `input.gamepad(i)` and bindings.

---

## Save and load

```typescript
import { SaveSystem, Transform } from '@emptysock/engine'

// Bind to a scene and an explicit list of save-aware components. Zero
// per-component save code, since components are plain serializable data.
const save = new SaveSystem(scene, [Transform, Health, Inventory], { variables: ctx.variables })
await save.save('slot-1')
await save.load('slot-1')
```

See `skills/33-save-system.md` for migrations and storage adapters, and `skills/07-save-localisation.md` for `LocalisationSystem`.

---

## Localisation

```typescript
// ctx.localisation is a LocalisationSystem provided to every scene
localisation.addTranslations('en', { greeting: 'Hello', 'hud.score': 'Score: {{score}}' })
localisation.setLocale('en')

localisation.t('greeting')                    // "Hello"
localisation.t('hud.score', { score: 42 })    // "Score: 42"
```

See `skills/07-save-localisation.md`.

---

## Timers, tweens and coroutines

```typescript
import { TweenManager, waitSeconds } from '@emptysock/engine'

// TweenManager timers (call tweens.update(dt) each frame; cancel on unload):
const tweens = new TweenManager()
const h = tweens.every(3.0, () => spawnEnemy())
tweens.after(2.0, () => openDoor())
tweens.to(sprite, { alpha: 0 }, { duration: 0.5, ease: 'cubicOut' })
// on unload: h.cancel()   (or tweens.killAll())

// Coroutine (stops automatically when the entity is destroyed):
entity.startCoroutine(function* bossSequence() {
  yield waitSeconds(1.0)
  boss.roar()
})
```

`waitFrames(n)` and `waitUntil(() => cond)` are the other yield helpers. See `skills/19-tweens.md`.

---

## Scene navigation

```typescript
await game.loadScene(MenuScene)               // replaces the current scene
await game.loadOverlay(PauseScene)            // stack an overlay on top
await game.unloadOverlay()                    // pop the most recent overlay
```

For timed fade/wipe/slide transitions use `SceneTransitionManager.transition(() => game.loadScene(Next), { effect: 'fade' })`; see `skills/21-post-process.md`.

---

## Story Graph (VNSystem)

`VNSystem` is in `@emptysock/vn`, not the core package.

```typescript
import { VNSystem, storyGraphToDialogueTree, type DialogueNode, type StoryGraph } from '@emptysock/vn'

const graph = (await (await fetch('assets/story/chapter1.storyGraph.json')).json()) as StoryGraph  // validate in real code
const tree = storyGraphToDialogueTree(graph)

const vn = new VNSystem(ctx.variables)   // share the game-wide VariableStore

// Register the listener BEFORE load()
vn.setListener({
  onNode(node: DialogueNode) {
    if (node.type === 'dialogue') showText(node.speaker, node.text)
  },
  onChoice(options) { showChoiceButtons(options) },   // options: { label, next, when? }[]
  onEnd() { hideDialogueBox() },
})

vn.load(tree)        // synchronous; fires onNode for the first node
vn.advance()         // move past a dialogue node
vn.selectOption(id)  // select a choice: id comes from option.next
```

Open the Story Graph panel via **Module → Story Graph** in the IDE. See `skills/08-story-graph.md`.

---

## Window management

`ctx.window` (from the scene lifecycle context) is the game's `WindowSystem`; it is also constructible as `new WindowSystem()`.

```typescript
await ctx.window.apply({ mode: 'windowed', width: GAME_WIDTH, height: GAME_HEIGHT, title: PROJECT_TITLE })

await ctx.window.setMode('fullscreen')   // 'windowed' | 'fullscreen' | 'borderless'
await ctx.window.setTitle('New Title')
await ctx.window.setSize(1920, 1080)
await ctx.window.setResizable(false)
await ctx.window.center()
```

`GAME_WIDTH`, `GAME_HEIGHT`, `PROJECT_TITLE`, `PROJECT_NAME`, and `DEBUG` are compile-time constants injected from project settings; use them freely, no import needed.

---

## Rules — never break these

| Wrong | Right |
|---|---|
| `import * as PIXI from 'pixi.js'` | Only `@emptysock/engine` |
| `setTimeout(() => fn(), 2000)` | `tweens.after(2.0, fn)` |
| `async onUpdate() {}` | Coroutines only |
| `entity.get(Sprite)!` | `entity.get(Sprite)?.prop` |
| `JSON.parse(x) as MyType` | `MySchema.parse(JSON.parse(x))` |
| `let x: any` | `let x: unknown` then narrow |
| Forgetting `scene.destroy(entity)` | Always destroy when done, same call whether pooled or not |
| Forgetting `handle.cancel()` | Always cancel timers on unload |
| Mutating a `networked()` field in place | Reassign a new value; see `skills/35-network-package.md` |
| Skipping `physics3d.destroy()` | Always call on unload; it leaks WASM otherwise |
| Engine-owned loading screen | Not a thing; build one as a scene if needed |
| Engine-owned splash screen | Not a thing; build one as a scene if needed |
