# EmptySock — Claude Code Instructions

You are building a game with **EmptySock** (`@emptysock/engine`) — a TypeScript-first
2D game engine (with optional 3D) that targets web, desktop, mobile, and Raspberry Pi.

Read this file fully before writing any code.

---

## Engine at a Glance

| Layer | Technology |
|---|---|
| 2D Renderer | PixiJS v8 (WebGL2 → WebGPU) |
| 3D Renderer | Three.js (optional, hybrid scenes) |
| Physics | Rapier2D WASM |
| Audio | Howler.js |
| Shell | Tauri v2 (desktop / mobile) |
| Language | TypeScript — strict mode, zero `any` |

---

## Absolute Rules — Never Break These

### Types
- Zero `any`. Use `unknown` with Zod parsing or explicit type guards.
- Zero `!` non-null assertions. Use `?.` and `?? defaultValue`.
- Zero `// @ts-ignore`. Use `// @ts-expect-error` with a reason comment if unavoidable.
- Every function needs an explicit return type.
- Use `import type` for type-only imports.
- All external data (JSON files, save data, API responses) must go through a **Zod schema** before use.

### Engine usage
- Never import PixiJS, Rapier, Howler, or Three.js directly. Use `@emptysock/engine` only.
- Never touch the DOM directly (`document.querySelector`, etc.). Use the EmptySock UI system.
- Never use `setTimeout` or `setInterval` in game logic. Use `Timer.after()` / `Timer.every()`.
- Never use `async/await` inside `onUpdate()` or `onFixedUpdate()`. Use coroutines (`function*`).
- Always call `entity.destroy()` when an entity is no longer needed.
- Always cancel timers in `onDestroy()` if they reference scene objects.

### Commits
- Conventional commit format: `feat(scope): message`, `fix(scope): message`, `refactor(scope): message`
- Commit after every working addition. Never commit broken code.

---

## Core Patterns

### Boot

```typescript
import { Engine } from '@emptysock/engine'
import { MenuScene } from './scenes/MenuScene'
import { GameScene } from './scenes/GameScene'

const engine = await Engine.create({
  scenes: { MenuScene, GameScene },
  startScene: 'MenuScene',
  gameSpeed: 60,
})
await engine.start()
```

### Scene

```typescript
import { Scene, type SceneConfig } from '@emptysock/engine'

export class GameScene extends Scene {
  static readonly config: SceneConfig = {
    renderMode: '2d',
    gameSpeed: 60,
    lighting: false,
  }

  override async onLoad(): Promise<void> {
    // Load assets, build entities. Awaited before scene renders.
  }

  override onUpdate(dt: number): void {
    // Every frame. dt = seconds since last frame.
    // No async/await here — use coroutines.
  }

  override onDestroy(): void {
    // Called before scene unloads. Cancel timers, remove listeners.
  }
}
```

### Entity & Components

```typescript
import { Sprite, PhysicsBody, CharacterController, Animator } from '@emptysock/engine'

const player = scene.createEntity('Player')
player.addComponent(Sprite, { texture: 'hero.png', anchor: { x: 0.5, y: 1.0 } })
player.addComponent(PhysicsBody, { shape: 'capsule', bodyType: 'dynamic' })
player.addComponent(CharacterController, { slopeAngle: 45, snapToGround: 0.5 })
player.addComponent(Animator, { spritesheet: 'hero.esanim', defaultClip: 'idle' })

// Access
const sprite = player.getComponent(Sprite)  // T | undefined
const body   = player.requireComponent(PhysicsBody) // T | throws

// Destroy
player.destroy()
```

### Input

```typescript
import { InputSystem } from '@emptysock/engine'

// In onLoad — create and attach:
private _input = new InputSystem()
this._input.attach()   // registers listeners on window

// In onUpdate(dt) — call flush() first, then read state:
this._input.flush()
if (this._input.isKeyDown('ArrowRight'))   { /* held every frame */ }
if (this._input.isKeyPressed('Space'))     { /* fired once on key-down */ }
if (this._input.isKeyReleased('Space'))    { /* fired once on key-up */ }

const x = this._input.mouseX   // mouse / pointer X
const y = this._input.mouseY   // mouse / pointer Y

// In onDestroy:
this._input.detach()
```

### Character movement (platformer)

```typescript
// In a custom Component's onUpdate(dt):
// Assumes this._input is an InputSystem instance attached in onLoad.
private vy = 0

override onUpdate(dt: number): void {
  this._input.flush()
  const ctrl  = this.entity.requireComponent(CharacterController)
  const anim  = this.entity.requireComponent(Animator)
  const h = (this._input.isKeyDown('ArrowRight') ? 1 : this._input.isKeyDown('ArrowLeft') ? -1 : 0)

  if (!ctrl.isGrounded()) this.vy += 980 * dt  // gravity
  else                    this.vy  = 0

  if (this._input.isKeyPressed('Space') && ctrl.isGrounded()) this.vy = -600

  ctrl.moveAndSlide({ x: h * 200 * dt, y: this.vy * dt })
  anim.play(Math.abs(h) > 0.1 ? 'run' : ctrl.isGrounded() ? 'idle' : 'fall')
  if (h !== 0) this.entity.scale.x = h > 0 ? 1 : -1
}
```

### Audio

```typescript
import { Audio } from '@emptysock/engine'

Audio.play('jump_sfx')
Audio.play('footstep', { volume: 0.6, spatial: true, position: entity.position })
Audio.music('level_theme', { loop: true, fade: 0.5 })
Audio.setGroupVolume('sfx', 0.8)
Audio.stopMusic({ fade: 0.5 })
```

### Camera

```typescript
import { Camera } from '@emptysock/engine'

Camera.follow(player, { lerp: 0.1, deadzone: { x: 80, y: 40 } })
Camera.shake({ intensity: 6, duration: 0.3 })
Camera.zoom(2.0, { duration: 0.4, ease: 'sineOut' })
Camera.fade({ to: 0x000000, duration: 0.5 })
Camera.unfade({ duration: 0.3 })
```

### Timers

```typescript
import { Timer } from '@emptysock/engine'

// Store handles for cleanup
private _spawnTimer: TimerHandle | null = null

override onLoad(): void {
  this._spawnTimer = Timer.every(3.0, () => { this.spawnEnemy() })
}

override onDestroy(): void {
  this._spawnTimer?.cancel()
}
```

### Coroutines

```typescript
// Sequences over time — use instead of async/await in game logic
entity.startCoroutine(function* boss_intro() {
  yield waitSeconds(1.0)
  dialogue.show('I have been waiting...')
  yield waitForDialogue()
  Camera.shake({ intensity: 12, duration: 0.5 })
  yield waitSeconds(0.5)
  boss.activate()
})
```

### Scene navigation

```typescript
import { SceneManager } from '@emptysock/engine'

SceneManager.load('GameScene')
SceneManager.transition('MenuScene', { effect: 'fade', duration: 0.4 })
SceneManager.push('PauseScene')   // overlay; previous scene pauses
SceneManager.pop()                 // return to previous scene
```

### Saving & loading

```typescript
import { SaveSystem } from '@emptysock/engine'
import { z } from 'zod'

// Always define a Zod schema — never skip validation
const SaveSchema = z.object({
  scene:     z.string(),
  score:     z.number(),
  inventory: z.array(z.string()),
  flags:     z.record(z.boolean()),
})
type SaveData = z.infer<typeof SaveSchema>

async function save(slot: string): Promise<void> {
  await SaveSystem.save(slot, { scene: 'Level2', score: 4200, inventory: [], flags: {} })
}

async function load(slot: string): Promise<SaveData> {
  const raw = await SaveSystem.load(slot)
  return SaveSchema.parse(raw.data) // always validate — throws on corrupt data
}
```

### Localisation

```typescript
import { LocalisationSystem } from '@emptysock/engine'
import { z } from 'zod'

// In onLoad — create instance and load locale files:
const localisation = new LocalisationSystem()
const TranslationMapSchema = z.record(z.string())
const enRaw = await (await fetch('assets/i18n/en.json')).json()
localisation.addTranslations('en', TranslationMapSchema.parse(enRaw))
localisation.setLocale('en')

// Translate:
const label = localisation.t('menu.start')                   // "Start Game"
const text  = localisation.t('hud.score', { score: 1200 })  // "Score: 1200"
const lang  = localisation.currentLocale                     // 'en'
```

### Tilemap

```typescript
import { TilemapSystem } from '@emptysock/engine'

const map = TilemapSystem.load('level1.esmap')
map.getLayer('Collision').enablePhysics()
const spawns = map.getLayer('Spawns').entities   // placed entity objects
```

### Physics events

```typescript
entity.onCollisionEnter((other, contact) => {
  if (other.hasTag('hazard')) player.takeDamage(10)
})
entity.onSensorEnter((other) => {
  if (other.hasTag('player')) openDoor()
})
```

### Dynamic lighting (requires `lighting: true` in SceneConfig)

```typescript
import { LightingSystem } from '@emptysock/engine'

private _lighting: LightingSystem | null = null

override onLoad(): void {
  this._lighting = new LightingSystem()
  this._lighting.attachFilter(this.stage)   // wire GPU filter — required
  this._lighting.setAmbient(0x111133, 0.08)

  this._lighting.addLight({
    id:          'torch-1',
    type:        'point',
    x:           300,
    y:           200,
    colour:      0xffaa44,
    intensity:   1.4,
    radius:      280,
    castShadows: false,
  })
}

override onUpdate(dt: number): void {
  if (this._lighting === null) return
  // Mutate position in-place — no remove/re-add needed
  const torch = this._lighting.lights.get('torch-1')
  if (torch !== undefined) {
    torch.x = this._player.position.x
    torch.y = this._player.position.y - 20
  }
  this._lighting.update(dt)   // upload to GPU uniforms
}
```

### Object pool (bullets, particles, enemies)

```typescript
import { ObjectPool } from '@emptysock/engine'

const pool = new ObjectPool(BulletEntity, { size: 200 })

function fire(): void {
  const b = pool.acquire()
  b.launch(player.position, aimDirection)
}

// Inside BulletEntity.onUpdate(dt):
if (this.isOffscreen()) pool.release(this)
```

---

## Common Mistakes — Never Do These

| Wrong | Right |
|---|---|
| `import * as PIXI from 'pixi.js'` | Use `@emptysock/engine` only |
| `import { Input } from '@emptysock/engine'` | `import { InputSystem } from '@emptysock/engine'` (instanced) |
| `Input.isPressed('Space')` | `input.isKeyPressed('Space')` on an `InputSystem` instance |
| `import { t, LocalisationSystem }` | `import { LocalisationSystem }` (no standalone `t` export) |
| `LocalisationSystem.setLocale('fr')` | `localisation.setLocale('fr')` on an instance |
| `setTimeout(() => spawnEnemy(), 2000)` | `Timer.after(2.0, () => spawnEnemy())` |
| `async onUpdate() { await fetch(...) }` | Preload in `onLoad()`, or use a coroutine |
| `entity.getComponent(Sprite)!` | `entity.getComponent(Sprite)?.prop` |
| `private _x!: SomeType` | `private _x: SomeType \| null = null` |
| `JSON.parse(raw) as MyType` | `MySchema.parse(JSON.parse(raw))` |
| `let x: any = getStuff()` | `let x: unknown = getStuff()` then narrow |
| `document.getElementById('canvas')` | EmptySock UI / canvas system |
| Forgetting `entity.destroy()` | Always destroy when done |
| Forgetting timer cleanup in `onDestroy` | Always cancel stored `TimerHandle`s |
| Forgetting `input.detach()` in `onDestroy` | Always detach `InputSystem` when done |

---

## Performance Quick Rules

- Keep draw calls under **50 per frame** when targeting Raspberry Pi 4 or older mobile.
- Use texture atlases — group sprites by shared texture to minimise draw call switches.
- Use object pools for anything that spawns frequently (bullets, particles, enemies).
- Avoid allocating objects inside `onUpdate()` — reuse with private fields.
- Check `Engine.gpuTier` to scale effects: `'potato' | 'low' | 'mid' | 'high' | 'ultra'`.
- Never enable soft shadows on `'potato'` or `'low'` tiers.

---

## Commit Format

```
feat(player): add wall-jump mechanic
feat(audio): implement dynamic music layering
fix(physics): character tunnels through thin platform at high speed
fix(save): corrupt slot crashes load screen
refactor(enemy): extract patrol logic into PatrolComponent
perf(particles): cap emitter count on low GPU tier
chore(assets): compress sprite atlas
```

---

## Further Reference

Full API, all systems, all methods:
→ `api-reference.json` (in this same repository under `ai/`)

Detailed skill guides per system:
→ `skills/` directory (in this same repository)
