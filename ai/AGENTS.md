# EmptySock — Agent Reference

Vendor-agnostic instructions for any AI agent working with the EmptySock game engine.
Use this as a system prompt prepend, context file, or project instruction.

---

## What is EmptySock?

EmptySock is a TypeScript game engine for 2D games (with optional 3D) that exports to:
Web (HTML5), Windows, macOS, Linux, Android, iOS, and Raspberry Pi.

The engine package is `@emptysock/engine`. It wraps PixiJS v8 (rendering),
Rapier2D WASM (physics), and Howler.js (audio) behind a clean, fully-typed API.
Agents never import those libraries directly — only `@emptysock/engine`.

---

## Non-Negotiable Rules

1. **TypeScript strict mode.** Zero `any`. Zero `!` non-null assertions. All external
   data parsed through Zod before use. Explicit return types on every function.

2. **No direct library imports.** Never import `pixi.js`, `@dimforge/rapier2d`, `howler`,
   or `three`. The engine re-exports everything through its own API.

3. **No DOM access.** Never call `document.querySelector`, `getElementById`, or similar.
   Use the engine's UI system.

4. **No setTimeout/setInterval in game logic.** Use `Timer.after()`, `Timer.every()`,
   or generator coroutines.

5. **No async/await in the game loop.** `onUpdate()` and `onFixedUpdate()` are synchronous.
   Use `function*` coroutines for sequenced async behaviour.

6. **Destroy what you create.** Every entity that is done must call `entity.destroy()`.
   Every `TimerHandle` must be cancelled in `onDestroy()` if it references scene state.

7. **Validate saves.** Never read save data as `unknown as MyType`. Always parse through
   a Zod schema before use. Saves can be corrupt.

8. **Conventional commits.** `feat(scope): what`, `fix(scope): what`. Commit after each
   working addition.

---

## Key API — the most-used surface

```typescript
// Engine boot
await Engine.create({ scenes: { MenuScene, GameScene }, startScene: 'MenuScene', gameSpeed: 60 })
await engine.start()

// Scene base class
class MyScene extends Scene {
  static readonly config: SceneConfig = { renderMode: '2d', gameSpeed: 60 }
  override async onLoad(): Promise<void> {}
  override onUpdate(dt: number): void {}       // dt = seconds
  override onDestroy(): void {}
}

// Entities & components
const e = scene.createEntity('Name')
e.addComponent(Sprite, { texture: 'file.png' })
e.addComponent(PhysicsBody, { shape: 'capsule', bodyType: 'dynamic' })
e.addComponent(CharacterController, { slopeAngle: 45 })
e.getComponent(Sprite)          // T | undefined
e.requireComponent(Sprite)      // T | throws
e.hasTag('enemy')               // boolean
e.destroy()

// Input
Input.isDown('ArrowRight')      // held
Input.isPressed('Space')        // just pressed this frame
Input.axis('Horizontal')        // -1..1
Input.pointer.position          // { x, y }

// Audio
Audio.play('sfx_id', { volume: 0.8 })
Audio.music('track_id', { loop: true, fade: 0.5 })
Audio.setGroupVolume('sfx', 0.8)

// Camera
Camera.follow(entity, { lerp: 0.1 })
Camera.shake({ intensity: 6, duration: 0.3 })
Camera.zoom(2.0, { duration: 0.4 })
Camera.fade({ to: 0x000000, duration: 0.5 })

// Scene navigation
SceneManager.load('SceneName')
SceneManager.transition('SceneName', { effect: 'fade', duration: 0.4 })
SceneManager.push('OverlayScene')
SceneManager.pop()

// Timers — store handle, cancel in onDestroy
const h = Timer.after(2.0, fn)
const h = Timer.every(0.5, fn)
h.cancel()

// Coroutines — async game sequences
entity.startCoroutine(function* () {
  yield waitSeconds(1.0)
  yield waitUntil(() => player.isGrounded())
  yield waitForEvent('boss_dead')
  SceneManager.transition('WinScene', { effect: 'iris' })
})

// Saves — always Zod validate
await SaveSystem.save('slot-1', data)
const raw = await SaveSystem.load('slot-1')
const safe = MySchema.parse(raw.data)

// Localisation
LocalisationSystem.setLocale('fr')
t('key.name')
t('key.score', { score: 100 })

// Tilemap
const map = TilemapSystem.load('level.esmap')
map.getLayer('Collision').enablePhysics()

// Physics events
entity.onCollisionEnter((other, contact) => {})
entity.onSensorEnter((other) => {})

// Tweens
Tween.to(entity, { x: 200 }, { duration: 0.5, ease: 'bounceOut' })

// Profiler
const stats = Profiler.getStats()  // fps, drawCalls, frameTime, memoryMB
```

---

## GPU Tiers

Check `Engine.gpuTier` to scale quality:

| Tier | Example hardware | Particles | Lights | Post-FX | FPS target |
|---|---|---|---|---|---|
| `potato` | Pi Zero, Pi 1 | 100 | 1 | Off | 30 |
| `low` | Pi 4 (4 GB), S9+ | 500 | 4 | 1 simple | 60 |
| `mid` | 2015 iGPU laptop | 2000 | 8 | 2 passes | 60 |
| `high` | GTX 1060, RX 580 | 5000 | 16 | Full | 60 |
| `ultra` | RTX 30+, M2+ | 10000 | 32 | Full | Unlocked |

---

## File Types

| Extension | Purpose |
|---|---|
| `.esscene` | Scene — entities, components, properties |
| `.esmap` | Tilemap — layers, tiles, collision |
| `.esanim` | Spritesheet animation clips |
| `.esprefab` | Reusable entity template |
| `.esvn` | Visual novel dialogue tree |
| `.esparticle` | Particle system definition |
| `.esui` | UI layout panel |
| `.esdata` | Game data (validated JSON) |

---

## Banned Patterns

```typescript
// BANNED — never do any of these
import * as PIXI from 'pixi.js'
import RAPIER from '@dimforge/rapier2d'
import { Howl } from 'howler'
setTimeout(fn, ms)
setInterval(fn, ms)
async onUpdate() {}
await anything_inside_onUpdate()
let x: any = something
entity.getComponent(Sprite)!
JSON.parse(data) as MyType           // always use Zod
document.querySelector(...)
document.getElementById(...)
window.localStorage.setItem(...)     // use SaveSystem
```

---

## Further Reference

This file is intentionally concise. For full documentation see:

- **skills/** — one file per system, full patterns and options
- **docs/** — getting started, core concepts, tutorials
- **ai/api-reference.json** — machine-readable full API for programmatic agent use
