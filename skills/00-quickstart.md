# EmptySock — Quickstart

Everything you need for a typical game task, in one page.

---

## Engine boot

```typescript
import { Engine } from '@emptysock/engine'

const engine = await Engine.create({
  scenes: { MenuScene, GameScene },
  startScene: 'MenuScene',
  gameSpeed: 60,
})
await engine.start()
```

---

## Scene skeleton

```typescript
import { Scene, type SceneConfig } from '@emptysock/engine'

export class GameScene extends Scene {
  static readonly config: SceneConfig = { renderMode: '2d', gameSpeed: 60 }

  override async onLoad(): Promise<void> {
    // build world here — awaited before first frame
  }

  override onStart(): void {
    // called once after onLoad resolves — safe to reference entities created in onLoad
  }

  override onUpdate(dt: number): void {
    // runs every frame — dt = seconds since last frame
    // no async/await here — use coroutines
  }

  override onFixedUpdate(dt: number): void {
    // runs at fixed timestep — use for physics / authoritative simulation
  }

  override onPause(): void {
    // called when another scene is pushed on top via SceneManager.push()
  }

  override onResume(): void {
    // called when that pushed scene is popped and this one becomes active again
  }

  override onDestroy(): void {
    // cancel timers, remove listeners, call physics3d.destroy()
  }
}
```

---

## Entities and components

```typescript
import { Sprite, PhysicsBody, CharacterController, Animator } from '@emptysock/engine'

const player = scene.createEntity('Player')
player.addComponent(Sprite, { texture: 'hero.png', anchor: { x: 0.5, y: 1.0 } })
player.addComponent(PhysicsBody, { shape: 'capsule', bodyType: 'dynamic' })
player.addComponent(CharacterController, { slopeAngle: 45 })
player.addComponent(Animator, { spritesheet: 'hero.esanim', defaultClip: 'idle' })

const sprite = player.getComponent(Sprite)          // T | undefined
const body   = player.requireComponent(PhysicsBody) // T | throws

// Transform — position, rotation (radians), scale:
player.position = { x: 100, y: 200 }
player.setPosition(100, 200)  // chainable
player.rotation = Math.PI / 4
player.scale = { x: 1.5, y: 1.5 }

// Events — on() returns an unsubscriber:
const unsub = player.on('hit', (data) => { console.log('hit', data) })
player.emit('hit', { damage: 10 })
unsub()  // stop listening

// Coroutines on the entity:
player.startCoroutine(function* () {
  yield waitSeconds(1.0)
  player.emit('ready')
})

// Destroy — removes from scene, destroys children, emits 'destroy':
player.destroy()
```

---

## 2D physics (sync, no init required)

```typescript
import { PhysicsBody, CharacterController } from '@emptysock/engine'

player.addComponent(PhysicsBody, { shape: 'capsule', bodyType: 'dynamic' })
player.addComponent(CharacterController, { slopeAngle: 45 })

// Declare in the class body: private _vy = 0
// In onUpdate (assuming this.input is an InputSystem attached in onLoad):
this.input.flush()
const ctrl = player.requireComponent(CharacterController)
if (!ctrl.isGrounded()) this._vy += 980 * dt
else this._vy = 0
if (ctrl.isGrounded() && this.input.isKeyPressed('Space')) this._vy = -600
const h = this.input.isKeyDown('ArrowRight') ? 1 : this.input.isKeyDown('ArrowLeft') ? -1 : 0
ctrl.moveAndSlide({ x: h * 200 * dt, y: this._vy * dt })
```

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
import { z } from 'zod'

const Schema = z.object({ scene: z.string(), score: z.number(), flags: z.record(z.boolean()) })
type Save = z.infer<typeof Schema>

const save = new SaveSystem()
const ok = save.save('slot-1', { scene: 'Level2', score: 4200, data: { flags: {} }, timestamp: Date.now(), playtime: 0 })
if (!ok) console.warn('save failed — storage unavailable')
const slot = save.load('slot-1')                     // SaveSlot | null
if (slot !== null) Schema.parse(slot.data)            // always validate
```

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
SceneManager.transition('MenuScene', { effect: 'fade', duration: 0.4 })
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
| `entity.getComponent(Sprite)!` | `entity.getComponent(Sprite)?.prop` |
| `JSON.parse(x) as MyType` | `MySchema.parse(JSON.parse(x))` |
| `let x: any` | `let x: unknown` then narrow |
| Forgetting `entity.destroy()` | Always destroy when done |
| Forgetting `handle.cancel()` | Always cancel timers in `onDestroy` |
| Skip `physics3d.destroy()` | Always call in `onDestroy` — leaks WASM |
| Engine-owned loading screen | Not a thing — build one as a scene if needed |
| Engine-owned splash screen | Not a thing — build one as a scene if needed |
