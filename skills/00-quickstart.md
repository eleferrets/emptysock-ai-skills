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

  override onUpdate(dt: number): void {
    // runs every frame — dt = seconds since last frame
    // no async/await here — use coroutines
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
player.destroy()
```

---

## 2D physics (sync, no init required)

```typescript
import { PhysicsBody, CharacterController } from '@emptysock/engine'

player.addComponent(PhysicsBody, { shape: 'capsule', bodyType: 'dynamic' })
player.addComponent(CharacterController, { slopeAngle: 45 })

// In onUpdate:
const ctrl = player.requireComponent(CharacterController)
if (ctrl.isGrounded() && Input.isPressed('Space')) ctrl.jump(600)
ctrl.moveAndSlide({ x: Input.axis('Horizontal') * 200 * dt, y: 0 })
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

class EnemyActor extends Actor {
  receive(msg: Message): void {
    if ((msg as any).type === 'TAKE_DAMAGE') { /* handle */ }
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
import { Input } from '@emptysock/engine'

if (Input.isDown('ArrowRight'))  { /* held every frame */ }
if (Input.isPressed('Space'))    { /* fired once on keydown */ }
if (Input.isReleased('Space'))   { /* fired once on keyup */ }

const h = Input.axis('Horizontal') // -1..1, keyboard or gamepad stick
const p = Input.pointer.position   // mouse or first touch position
```

---

## Platformer character (copy-paste start)

```typescript
private vy = 0

override onUpdate(dt: number): void {
  const ctrl = this.entity.requireComponent(CharacterController)
  const anim = this.entity.requireComponent(Animator)
  const h    = Input.axis('Horizontal')

  if (!ctrl.isGrounded()) this.vy += 980 * dt
  else                    this.vy  = 0

  if (Input.isPressed('Space') && ctrl.isGrounded()) this.vy = -600

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

```typescript
import { Camera } from '@emptysock/engine'

Camera.follow(player, { lerp: 0.1, deadzone: { x: 80, y: 40 } })
Camera.shake({ intensity: 6, duration: 0.3 })
Camera.zoom(2.0, { duration: 0.4, ease: 'sineOut' })
Camera.fade({ to: 0x000000, duration: 0.5 })
```

---

## Save and load

```typescript
import { SaveSystem } from '@emptysock/engine'
import { z } from 'zod'

const Schema = z.object({ scene: z.string(), score: z.number(), flags: z.record(z.boolean()) })
type Save = z.infer<typeof Schema>

await SaveSystem.save('slot-1', { scene: 'Level2', score: 4200, flags: {} })
const raw  = await SaveSystem.load('slot-1')
const data = Schema.parse(raw.data) // always validate — throws on corrupt
```

---

## Localisation

```typescript
import { i18n } from '@emptysock/engine'

await i18n.load('en', () => import('./locales/en.json'))
i18n.setLocale('en')
i18n.t('greeting')           // → "Hello"
i18n.t('score', { n: 42 })  // → "Score: 42"
```

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
import { VNSystem, type VNNode, type VNDialogueNode, type VNChoiceNode } from '@emptysock/engine'

// In onLoad:
const vn = new VNSystem()
await vn.loadScript('assets/story/chapter1.vnscript')   // exported from Story Graph panel

vn.onNode((node: VNNode) => {
  if (node.type === 'dialogue') {
    const d = node as VNDialogueNode
    showText(d.speaker, d.text)
  } else if (node.type === 'choice') {
    const c = node as VNChoiceNode
    showChoiceButtons(c.options.map((o) => o.label))
  }
})

vn.play()
vn.advance()          // move past a dialogue node
vn.choose(0)          // select first choice option
vn.setVariable('flag', true)
vn.jumpToNode('id')   // resume from a saved node id
vn.destroy()          // in onDestroy
```

Open the Story Graph panel via **Module → Story Graph** in the IDE. Export the graph as `.vnscript` JSON.

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
