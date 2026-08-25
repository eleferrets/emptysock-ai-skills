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
    // cancel timers, remove listeners
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

// Read
const sprite = player.getComponent(Sprite)          // T | undefined
const body   = player.requireComponent(PhysicsBody) // T | throws

// Cleanup
player.destroy()
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

## Timers

```typescript
import { Timer, type TimerHandle } from '@emptysock/engine'

private _spawn: TimerHandle | null = null

override onLoad(): void {
  this._spawn = Timer.every(3.0, () => this.spawnEnemy())
}

override onDestroy(): void {
  this._spawn?.cancel() // always cancel in onDestroy
}
```

---

## Coroutines

```typescript
entity.startCoroutine(function* boss_sequence() {
  yield waitSeconds(1.0)
  boss.roar()
  yield waitForAnimation(boss)
  Camera.shake({ intensity: 10, duration: 0.4 })
  yield waitSeconds(0.3)
  boss.startAttacking()
})
```

---

## Scene navigation

```typescript
import { SceneManager } from '@emptysock/engine'

SceneManager.load('GameScene')
SceneManager.transition('MenuScene', { effect: 'fade', duration: 0.4 })
SceneManager.push('PauseScene')   // overlay; previous pauses
SceneManager.pop()                 // return
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

## Tilemap

```typescript
import { TilemapSystem } from '@emptysock/engine'

const map    = TilemapSystem.load('level1.esmap')
map.getLayer('Collision').enablePhysics()
const spawns = map.getLayer('Spawns').entities
```

---

## Tweens

```typescript
import { Tween } from '@emptysock/engine'

Tween.to(entity, { x: 400 }, { duration: 0.5, ease: 'bounceOut' })
Tween.to(sprite, { alpha: 0 }, { duration: 0.3, onComplete: () => entity.destroy() })
```

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
