# EmptySock — Quickstart

Everything you need for a typical game task, in one page. For the full story on the entity/component/scene core, see `skills/32-ecs-core.md`.

---

## Engine boot and scene skeleton

```typescript
import { Game, defineScene } from '@emptysock/engine'

const GameScene = defineScene({
  async onLoad(scene, { actors, physics, input, audio }) {
    // build world here — awaited before first frame. actors/physics already
    // exist for you; Game created them, never `new ActorSystem()` yourself.
  },
  onUpdate(dt) {
    // runs every frame, after physics/actor/collision, before render.
    // no async here — it's a type error. Use entity.startCoroutine() instead.
  },
  onUnload(scene) {
    // called before this scene's actors/physics are torn down (automatically)
  },
})

const game = new Game()
await game.loadScene(GameScene)
// each frame, from your host's render loop: game.update(dt)
```

`game.loadOverlay(HudScene)` stacks an independent scene on top (HUD, pause menu) that survives a main-scene reload underneath it. `game.services` is a typed registry for process-global state — see `skills/32-ecs-core.md`.

---

## Entities and components

Entities are handles, not classes to subclass. Components are plain data from `defineComponent`.

```typescript
import { defineComponent, Transform, Sprite, PhysicsBody } from '@emptysock/engine'

const player = scene.spawn('Player')
player.add(Transform, { x: 0, y: 0 })
player.add(Sprite, { texturePath: 'hero.png', anchorX: 0.5, anchorY: 1.0 })
player.add(PhysicsBody, { shape: 'capsule', type: 'dynamic' })

player.has(PhysicsBody)              // boolean
const body = player.get(PhysicsBody) // T | undefined — proxy onto live data

// Coroutines on the entity — still the escape hatch for multi-frame work:
player.startCoroutine(function* () {
  yield waitSeconds(1.0)
})

// Destroy — same call whether or not the entity was spawned pooled:
scene.destroy(player)
```

Define your own components with `defineComponent(name, () => defaults, options?)` — see `skills/32-ecs-core.md` for the full rules (serializable fields only, name is the real identity across hot-reload). For touching many entities at once, use `scene.each(Transform, PhysicsBody, (t, b, entity) => {...})` instead of per-entity `.get()` — same objects, just the bulk path.

---

## Rendering (RenderPipeline)

Attaching `Transform` + `Sprite` to an entity is the entire contract for "this shows up on screen" — no manual PixiJS wiring. See `skills/23-rendering.md` for tilemaps, layers, and custom texture loading.

```typescript
import { RenderPipeline, Transform, Sprite } from '@emptysock/engine'

// In onLoad:
private _render = new RenderPipeline()
await this._render.init({ width: 1280, height: 720 })
document.body.appendChild(this._render.canvas)

// Anything with Transform + Sprite is drawn automatically:
const player = scene.createEntity('Player')
player.addComponent(new Transform({ x: 100, y: 200 }))
player.addComponent(new Sprite({ texturePath: 'hero.png', layer: 'foreground', depth: 10 }))

// Call once per frame, after game logic:
override onUpdate(dt: number): void {
  this._render.renderFrame(this)
}

// In onDestroy:
this._render.destroy()
```

---

## 2D physics

```typescript
import { PhysicsBody, CharacterController } from '@emptysock/engine'

player.add(PhysicsBody, { shape: 'capsule', type: 'dynamic' })
player.add(CharacterController, { slopeAngle: 45 })

// Declare in the class body: private _vy = 0
// In onUpdate (input is the frozen snapshot from game.input, passed in via SceneLifecycle):
const ctrl = player.get(CharacterController)
if (ctrl !== undefined) {
  if (!ctrl.isGrounded()) this._vy += 980 * dt
  else this._vy = 0
  if (ctrl.isGrounded() && input.isPressed('Space')) this._vy = -600
  const h = input.isDown('ArrowRight') ? 1 : input.isDown('ArrowLeft') ? -1 : 0
  ctrl.moveAndSlide({ x: h * 200 * dt, y: this._vy * dt })
}

// Collision/sensor callbacks are a direct property assignment — assigning IS registering:
const body = player.get(PhysicsBody)
if (body !== undefined) {
  body.onCollisionEnter = (other, contact) => {
    if (contact.impactForce > 50) console.log('Ouch.')
  }
  body.onSensorEnter = (other) => { /* trigger volume entered */ }
}
```

Bodies collide with everything by default — collision groups are opt-in tuning, not a setup requirement.

---

## 3D physics (Rapier3D — must await init)

```typescript
import { PhysicsSystem3D } from '@emptysock/engine'

// In onLoad:
const physics = new PhysicsSystem3D()
await physics.init({ x: 0, y: -9.81, z: 0 })

const box = physics.addBody({
  bodyType: 'dynamic',
  shape: 'box',
  halfExtents: { x: 0.5, y: 0.5, z: 0.5 },
  position: { x: 0, y: 5, z: 0 },
})

// In onUpdate:
physics.update(dt)
const pos = box.getPosition() // { x, y, z }

// In onDestroy — REQUIRED, frees WASM memory:
physics.destroy()
```

---

## Actor Model (message-driven logic)

```typescript
import { Actor, ActorSystem, type Message } from '@emptysock/engine'

// Define a typed message union for this actor:
type TakeDamageMsg = { type: 'TAKE_DAMAGE'; amount: number }
type EnemyMsg = TakeDamageMsg   // extend with more variants as needed

class EnemyActor extends Actor {
  receive(msg: Message): void {
    const m = msg as EnemyMsg
    if (m.type === 'TAKE_DAMAGE') { /* handle m.amount */ }
  }
  update(dt: number): void { /* per-frame AI */ }
}

const system = new ActorSystem()
system.register(new EnemyActor('enemy-1'))
system.send('enemy-1', { type: 'TAKE_DAMAGE', amount: 25 })
system.update(dt)   // flush mailboxes then run update() on all actors
system.destroy()    // in onDestroy
```

In a normal scene you don't construct `ActorSystem` yourself at all — `Game.loadScene()` creates and destroys one per scene automatically and hands it to you as `actors` in `onLoad`/`onUpdate`. The snippet above is what that automatic system does under the hood, useful to know if you're debugging message ordering.

---

## Input

```typescript
import { InputSystem } from '@emptysock/engine'

// In onLoad — create once, attach listeners:
private input = new InputSystem()
this.input.attach()

// In onUpdate — flush first, then read state:
this.input.flush()
if (this.input.isKeyDown('ArrowRight'))   { /* held every frame */ }
if (this.input.isKeyPressed('Space'))     { /* fired once on keydown */ }
if (this.input.isKeyReleased('Space'))    { /* fired once on keyup */ }

const mx = this.input.mouseX   // pointer X
const my = this.input.mouseY   // pointer Y

// In onDestroy:
this.input.detach()
```

---

## Platformer character (copy-paste start)

```typescript
private vy = 0
private input = new InputSystem()

override onLoad(): void { this.input.attach() }
override onDestroy(): void { this.input.detach() }

override onUpdate(dt: number): void {
  this.input.flush()
  const ctrl = this.entity.requireComponent(CharacterController)
  const anim = this.entity.requireComponent(Animator)
  const h    = this.input.isKeyDown('ArrowRight') ? 1 : this.input.isKeyDown('ArrowLeft') ? -1 : 0

  if (!ctrl.isGrounded()) this.vy += 980 * dt
  else                    this.vy  = 0

  if (this.input.isKeyPressed('Space') && ctrl.isGrounded()) this.vy = -600

  ctrl.moveAndSlide({ x: h * 200 * dt, y: this.vy * dt })

  if (Math.abs(h) > 0.1) this.entity.scale.x = h > 0 ? 1 : -1
  anim.play(Math.abs(h) > 0.1 ? 'run' : ctrl.isGrounded() ? 'idle' : 'fall')
}
```

---

## Particles

```typescript
import { ParticleEmitter } from '@emptysock/engine'

const entity = scene.createEntity('sparks')
const emitter = entity.addComponent(ParticleEmitter, {
  rate: 30, lifetime: 1.5, speed: 120, spread: 45, count: 100,
  textureName: 'fx/spark.png',   // optional, defaults to white square
})
emitter.start()           // continuous emission
emitter.burst(24)         // one-shot, ignores rate
emitter.stop()            // stop new particles; existing ones finish
// in onDestroy: emitter.stop()
```

---

## Audio

```typescript
import { Audio } from '@emptysock/engine'

Audio.play('jump_sfx')
Audio.play('footstep', { volume: 0.6, spatial: true, position: entity.position })
Audio.music('level_theme', { loop: true, fade: 0.5 })
Audio.setGroupVolume('sfx', 0.8)
Audio.stopMusic({ fade: 0.5 })
```

---

## Camera

`CameraSystem` is an instance you create per scene, then attach to the PixiJS stage.

```typescript
import { CameraSystem } from '@emptysock/engine'
import type { CameraBounds } from '@emptysock/engine'

const camera = new CameraSystem()
camera.attach(stage)                         // pass your PixiJS Container
camera.setViewSize(1280, 720)

// Follow a moving entity (smooth)
camera.setFollow(() => player.position)
camera.setLerpFactor(0.1)                   // 0 = no movement, 1 = instant

// Snap / move without follow
camera.snapTo(0, 0)                         // instant
camera.moveTo(500, 300)                     // smooth to target

// Zoom
camera.zoomTo(2.0)                          // smooth
camera.snapZoom(1.0)                        // instant

// Screen shake
camera.shake(6, 0.3)                        // intensity px, duration seconds

// Clamp to world bounds (accounts for zoom)
const bounds: CameraBounds = { minX: 0, minY: 0, maxX: 4000, maxY: 2000 }
camera.setBounds(bounds)
camera.setBounds(null)                      // remove clamping

// Coordinate conversion
const worldPos = camera.screenToWorld(mouseX, mouseY)
const screenPos = camera.worldToScreen(enemy.x, enemy.y)

// Call every frame
camera.update(dt)

// Clean up with scene
camera.destroy()
```

## Gamepad

```typescript
import { GamepadSystem } from '@emptysock/engine'

const gamepad = new GamepadSystem()

// In onUpdate(dt):
gamepad.update()
const state = gamepad.getState(0)          // pad index 0 | null when disconnected

if (gamepad.isButtonPressed(0, 0))  jump() // A button, just pressed
if (gamepad.isButtonDown(0, 2))     attack()
if (gamepad.isButtonReleased(0, 0)) land()
const leftX = state?.axes[0] ?? 0          // left stick X, -1..1

// Rumble
gamepad.rumble(0, 0.8, 200)                // padIndex, intensity 0-1, ms
gamepad.rumbleDual(0, { weakMagnitude: 0.3, strongMagnitude: 0.8, duration: 300 })

gamepad.destroy()
```

---

## Save and load

```typescript
import { SaveSystem } from '@emptysock/engine'

// Bind to a scene and an explicit list of save-aware components — zero
// per-component save code required, since components are plain serializable data.
const save = new SaveSystem(scene, [Transform, Health, Inventory])
await save.save('slot-1')
await save.load('slot-1')
```

See `skills/33-save-system.md` for migrations and storage adapters, and `skills/07-save-localisation.md` for `LocalisationSystem`.

---

## Localisation

```typescript
import { LocalisationSystem } from '@emptysock/engine'
import { z } from 'zod'

// In onLoad — create instance, load locale file:
const localisation = new LocalisationSystem()
const TranslationMapSchema = z.record(z.string())
const raw = await (await fetch('assets/i18n/en.json')).json()
localisation.addTranslations('en', TranslationMapSchema.parse(raw))
localisation.setLocale('en')

// Translate:
localisation.t('greeting')                    // → "Hello"
localisation.t('hud.score', { score: 42 })   // → "Score: 42"
```

Locale JSON files live at `assets/i18n/[locale].json`. See `skills/07-save-localisation.md` for the full format.

---

## Timers and coroutines

```typescript
import { Timer, waitSeconds, waitForAnimation } from '@emptysock/engine'

// Timer (cancel in onDestroy):
const h = Timer.every(3.0, () => this.spawnEnemy())
// in onDestroy: h.cancel()

// Coroutine:
entity.startCoroutine(function* boss_sequence() {
  yield waitSeconds(1.0)
  boss.roar()
  yield waitForAnimation(boss)
  boss.startAttacking()
})
```

---

## Scene navigation

```typescript
import { SceneManager } from '@emptysock/engine'

SceneManager.load('GameScene')
SceneManager.transition('MenuScene', { duration: 0.4 })
SceneManager.push('PauseScene')
SceneManager.pop()
```

---

## Story Graph (VNSystem)

```typescript
import {
  VNSystem,
  storyGraphToDialogueTree,
  type DialogueNode,
  type StoryGraph,
} from '@emptysock/engine'

// In onLoad (register callbacks BEFORE load):
const response = await fetch('assets/story/chapter1.storyGraph.json')
import { z } from 'zod'
const StoryGraphSchema = z.object({
  nodes: z.array(z.record(z.unknown())),
  edges: z.array(z.record(z.unknown())),
  startNodeId: z.string(),
})
const raw = await (await fetch('assets/story/chapter1.storyGraph.json')).json()
const graph = StoryGraphSchema.parse(raw) as StoryGraph
const tree = storyGraphToDialogueTree(graph)

const vn = new VNSystem()

// on* methods return an unsubscribe function — call it in onDestroy
const offNode = vn.onNode((node: DialogueNode) => {
  if (node.type === 'dialogue') {
    showText(node.speaker, node.text)
  } else if (node.type === 'choice') {
    showChoiceButtons(node.options)   // options: Array<{ label, next }>
  }
})
const offEnd = vn.onEnd(() => { hideDialogueBox() })

vn.load(tree)        // synchronous — fires onNode for first node immediately
vn.advance()         // move past a dialogue node
vn.selectOption(id)  // select a choice — id comes from option.next
// In onDestroy: offNode(); offEnd()
```

Open the Story Graph panel via **Module → Story Graph** in the IDE. Export the graph as `.storyGraph.json`.

---

## Window management

```typescript
import { windowSystem } from '@emptysock/engine'

// Apply from project settings on startup:
await windowSystem.apply({ mode: 'windowed', width: GAME_WIDTH, height: GAME_HEIGHT, title: PROJECT_TITLE })

// Runtime changes:
await windowSystem.setMode('fullscreen')   // 'windowed' | 'fullscreen' | 'borderless'
await windowSystem.setTitle('New Title')
await windowSystem.setSize(1920, 1080)
await windowSystem.setResizable(false)
await windowSystem.center()
```

`GAME_WIDTH`, `GAME_HEIGHT`, `PROJECT_TITLE`, `PROJECT_NAME`, and `DEBUG` are compile-time constants
injected from project settings — use them freely, no import needed.

---

## Rules — never break these

| Wrong | Right |
|---|---|
| `import * as PIXI from 'pixi.js'` | Only `@emptysock/engine` |
| `setTimeout(() => fn(), 2000)` | `Timer.after(2.0, fn)` |
| `async onUpdate() {}` | Coroutines only |
| `entity.get(Sprite)!` | `entity.get(Sprite)?.prop` |
| `JSON.parse(x) as MyType` | `MySchema.parse(JSON.parse(x))` |
| `let x: any` | `let x: unknown` then narrow |
| Forgetting `scene.destroy(entity)` | Always destroy when done — same call whether pooled or not |
| Forgetting `handle.cancel()` | Always cancel timers in `onDestroy` |
| Mutating a `networked()` field in place | Reassign a new value — see `skills/35-network-package.md` |
| Skip `physics3d.destroy()` | Always call in `onDestroy` — leaks WASM |
| Engine-owned loading screen | Not a thing — build one as a scene if needed |
| Engine-owned splash screen | Not a thing — build one as a scene if needed |
