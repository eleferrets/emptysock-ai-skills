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
- Never use `setTimeout` or `setInterval` in game logic. Use `TweenManager.after()` / `TweenManager.every()` on a per-scene instance.
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
import { Transform, Sprite, PhysicsBody, CharacterController, Animator } from '@emptysock/engine'

const player = scene.createEntity('Player')
player.addComponent(new Transform({ x: 0, y: 0 }))
player.addComponent(new Sprite({ texturePath: 'hero.png', anchorX: 0.5, anchorY: 1.0 }))
player.addComponent(new PhysicsBody({ shape: 'capsule', bodyType: 'dynamic' }))
player.addComponent(new CharacterController({ slopeAngle: 45, snapToGround: 0.5 }))
player.addComponent(new Animator({ spritesheet: 'hero.esanim', defaultClip: 'idle' }))

// Access — built-in components expose a typed `.TYPE` token for getComponent/requireComponent
const sprite = player.getComponent(Sprite.TYPE)  // Sprite | undefined
const body   = player.requireComponent(PhysicsBody.TYPE) // PhysicsBody | throws

// Destroy
player.destroy()
```

### Rendering (RenderPipeline)

Attaching `Transform` + `Sprite` to an entity is the entire contract for "this shows up on screen" — `RenderPipeline` finds every such entity each frame, keeps a synced sprite, and draws it. There is no manual PixiJS wiring, ever.

```typescript
import { RenderPipeline, Transform, Sprite } from '@emptysock/engine'

// In onLoad:
private _render = new RenderPipeline()
await this._render.init({ width: 1280, height: 720 })
document.body.appendChild(this._render.canvas)

// Call once per frame, after game logic:
override onUpdate(dt: number): void {
  this._render.renderFrame(this)
}

// In onDestroy:
this._render.destroy()
```

`RenderPipeline.mountTilemap(tilemap, layerName?, autoTileSystem?)` draws a `Tilemap`'s tiles as real textured sprites — `Tilemap` has no rendering of its own. See `skills/23-rendering.md` for the full API.

Pair `RenderPipeline` with `ViewportSystem` in every game that runs on more than one screen size — `RenderPipeline` draws, `ViewportSystem` scales what it drew to fit the container:

```typescript
import { ViewportSystem } from '@emptysock/engine'

private _viewport = new ViewportSystem()

override onLoad(): void {
  this._viewport.init(
    { designWidth: 1280, designHeight: 720, scaleMode: 'fit' },
    { renderTarget: this._render, cameraSystem: this._camera },
  )
}

override onDestroy(): void {
  this._viewport.destroy()
}
```

See `skills/24-viewport-system.md` for scale modes, safe-area insets, and GPU-tier render defaults.

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

For touch/mouse/pen gestures (tap, long-press, swipe, pinch), use `PointerSystem` alongside `InputSystem`/`GamepadSystem` rather than hand-rolling gesture detection — see `skills/25-pointer-system.md`. To let players remap controls, wrap `InputSystem`/`GamepadSystem` in `InputBindings` and query named actions (`bindings.isActionActive('jump')`) instead of raw key codes — see `skills/28-accessibility-debugging.md`.

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
import { AudioSystem } from '@emptysock/engine'

AudioSystem.play('jump_sfx')
AudioSystem.play('footstep', { volume: 0.6, spatial: true, position: entity.position })
AudioSystem.music('level_theme', { loop: true, fade: 0.5 })
AudioSystem.setGroupVolume('sfx', 0.8)
AudioSystem.stopMusic({ fade: 0.5 })

// Mixer: duck the music bus while dialogue plays, then release it
AudioSystem.duck('music', 0.3, 0.2)
AudioSystem.endDuck('music')

// Mixer: named volume snapshots (e.g. "combat", "explore")
AudioSystem.defineSnapshot('combat', { music: 0.4, sfx: 1, ui: 1, voice: 0.8 })
AudioSystem.transitionToSnapshot('combat', 0.5)
```

### Camera

```typescript
import { CameraSystem } from '@emptysock/engine'

// Create one CameraSystem per scene
private _camera = new CameraSystem()

override onLoad(): void {
  this._camera.attach(this.stage)            // wire to PixiJS container — required
  this._camera.setFollow(() => player.position)
  this._camera.setLerpFactor(0.1)            // 0.05 = slow drift, 1.0 = instant
  this._camera.setBounds({ minX: 0, minY: 0, maxX: 3200, maxY: 900 })
}

override onUpdate(dt: number): void {
  this._camera.update(dt)                    // must be called every frame
  // On hit:
  this._camera.shake(6, 0.3)               // intensity px, duration seconds
  this._camera.zoomTo(2.0)                 // smooth zoom toward target
  // Convert coordinates:
  const world = this._camera.screenToWorld(clickX, clickY)
  const screen = this._camera.worldToScreen(entity.position.x, entity.position.y)
}

override onDestroy(): void {
  this._camera.destroy()
}
```

### Timers

```typescript
import { TweenManager } from '@emptysock/engine'

// Create one TweenManager per scene; call update(dt) in onUpdate
private _tweens = new TweenManager()

override onLoad(): void {
  this._tweens.every(3.0, () => { this.spawnEnemy() })
}

override onUpdate(dt: number): void {
  this._tweens.update(dt)
}

override onDestroy(): void {
  this._tweens.destroy()   // cancels all pending tweens and timers
}
```

### Coroutines

```typescript
import { waitSeconds, waitUntil } from '@emptysock/engine'

// Sequences over time — use instead of async/await in game logic
entity.startCoroutine(function* boss_intro() {
  yield waitSeconds(1.0)
  dialogue.show('I have been waiting...')
  yield waitUntil(() => !dialogue.isVisible())
  this._camera.shake(12, 0.5)
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

`transition()`'s `effect` (`'none' | 'fade' | 'wipe' | 'slide'`) is only timed and tracked by `SceneManager` — it never touches pixi/DOM. To actually paint it, attach a `PostProcessSystem` once and pass it into your render call each frame:

```typescript
import { SceneManagerInstance, PostProcessSystem, RenderPipeline } from '@emptysock/engine'

private _postProcess = new PostProcessSystem()

override onLoad(): void {
  SceneManagerInstance.attachPostProcess(this._postProcess)
}

override onUpdate(dt: number): void {
  this._postProcess.update(dt)
  this._render.renderFrame(this, this._postProcess)   // paints the transition overlay too
}
```

### Saving & loading

```typescript
import { SaveSystem, type GameSaveSlot } from '@emptysock/engine'

// No schema given → SaveSystem uses the default GameSaveSlot shape:
// { id, scene, data, timestamp, playtime }. save()/load() are synchronous.
private _save = new SaveSystem()

function saveGame(slot: string): void {
  this._save.save(slot, { scene: 'Level2', data: { score: 4200, inventory: [], flags: {} } })
}

function loadGame(slot: string): GameSaveSlot | null {
  return this._save.load(slot)   // already validated against the schema — never throws
}
```

Pass a custom Zod schema as the second constructor argument (`new SaveSystem(prefix, schema)`) for a save shape that doesn't fit `{ scene, data, timestamp, playtime }` — the schema must require an `id: string` field, which `save()` fills in automatically. See `skills/07-save-localisation.md`.

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

Register real collision/sensor callbacks directly on the `PhysicsBody` you already hold — no second lookup:

```typescript
import { PhysicsBody } from '@emptysock/engine'

const hazard = player.requireComponent(PhysicsBody)
hazard.onCollisionEnter((other, contact) => {
  if (contact.impactForce > 50) player.takeDamage(10)
})

const sensor = door.requireComponent(PhysicsBody)
sensor.onSensorEnter((other) => {
  openDoor()
})
sensor.onSensorExit((other) => { closeDoor() })
sensor.onSensorStay((other) => { /* fires every step while inside */ })
```

`entity.onCollisionEnter()` / `entity.onSensorEnter()` still work (an older entity-event path that fires side by side with the `PhysicsBody` callbacks above) but prefer the `PhysicsBody` methods in new code.

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

Engine documentation (Unity/Unreal-style layout, lives in the engine repository):

| Path | Contents |
|---|---|
| `docs/getting-started/` | Install, first project, first run |
| `docs/guides/` | Physics, NavMesh, multiplayer, exports, and more |
| `docs/reference/` | Every class, method, property, and type |
| `docs/tutorials/` | Complete games built from scratch |
