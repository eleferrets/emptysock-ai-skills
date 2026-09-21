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

4. **No setTimeout/setInterval in game logic.** Use `tweens.after()`, `tweens.every()`
   on a per-scene `TweenManager` instance, or generator coroutines.

5. **No async/await in the game loop.** `onUpdate()` and `onFixedUpdate()` are synchronous.
   Use `function*` coroutines for sequenced async behaviour.

6. **Destroy what you create.** Every entity that is done must call `entity.destroy()`.
   Call `tweens.destroy()` in `onDestroy()` to cancel all pending timers and tweens.

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
e.addComponent(new Transform({ x: 0, y: 0 }))
e.addComponent(new Sprite({ texturePath: 'file.png' }))
e.addComponent(new PhysicsBody({ shape: 'capsule', bodyType: 'dynamic' }))
e.addComponent(new CharacterController({ slopeAngle: 45 }))
e.getComponent(Sprite.TYPE)          // Sprite | undefined — built-ins expose a typed `.TYPE` token
e.requireComponent(Sprite.TYPE)      // Sprite | throws
e.hasTag('enemy')               // boolean
e.destroy()

// Rendering — Transform + Sprite is the entire contract for "this shows up on screen"
const render = new RenderPipeline()                    // in onLoad
await render.init({ width: 1280, height: 720 })
document.body.appendChild(render.canvas)
render.renderFrame(scene)                               // once per frame, after game logic
render.mountTilemap(tilemap)                             // draws real tile sprites
render.destroy()                                         // in onDestroy

// Viewport — pair with RenderPipeline for design-resolution scaling across screen sizes
const viewport = new ViewportSystem()                    // in onLoad
viewport.init({ designWidth: 1280, designHeight: 720, scaleMode: 'fit' }, { renderTarget: render, cameraSystem: camera })
viewport.destroy()                                       // in onDestroy

// Input — instanced; create in onLoad, detach in onDestroy
// const input = new InputSystem(); input.attach();
// In onUpdate: input.flush() first, then:
input.isKeyDown('ArrowRight')   // held every frame
input.isKeyPressed('Space')     // true only on the frame the key went down
input.isKeyReleased('Space')    // true only on the frame the key came up
input.mouseX / input.mouseY     // pointer position

// PointerSystem — unified mouse/touch/pen + gestures; pair with InputSystem/GamepadSystem
const pointer = new PointerSystem()                      // in onLoad
pointer.attach()
pointer.onGesture((g) => { if (g.type === 'swipe') { /* ... */ } })
// In onUpdate: pointer.update() to poll for long-press
pointer.destroy()                                        // in onDestroy

// InputBindings — named actions over InputSystem/GamepadSystem, remappable and persisted via SaveSystem
const bindings = new InputBindings(input, { jump: [{ kind: 'key', code: 'Space' }] }, gamepad)
bindings.isActionActive('jump')
bindings.rebind('jump', [{ kind: 'key', code: 'ArrowUp' }])

// Audio — static class, call directly
AudioSystem.play('sfx_id', { volume: 0.8 })
AudioSystem.music('track_id', { loop: true, fade: 0.5 })
AudioSystem.setGroupVolume('sfx', 0.8)
AudioSystem.duck('music', 0.3, 0.2)                      // duck a bus, e.g. during dialogue
AudioSystem.transitionToSnapshot('combat', 0.5)          // named volume snapshot

// Camera — instanced; create in onLoad, update in onUpdate, destroy in onDestroy
const camera = new CameraSystem()
camera.attach(stage)                      // wire to PixiJS container
camera.setFollow(() => entity.position)   // track a moving target
camera.setLerpFactor(0.1)                // smoothing (0.05 = slow, 1.0 = instant)
camera.shake(6, 0.3)                     // intensity px, duration seconds
camera.zoomTo(2.0)                       // smooth zoom to scale factor
camera.setBounds({ minX: 0, minY: 0, maxX: 3200, maxY: 900 })
camera.update(dt)                        // call every frame in onUpdate

// Scene navigation
SceneManager.load('SceneName')
SceneManager.transition('SceneName', { effect: 'fade', duration: 0.4 })  // 'none'|'fade'|'wipe'|'slide'
SceneManager.push('OverlayScene')
SceneManager.pop()
// transition()'s effect is only timed/tracked here — it never touches pixi/DOM.
// Call SceneManagerInstance.attachPostProcess(postProcess) once (a PostProcessSystem
// instance), then pass that instance into RenderPipeline.renderFrame(scene, postProcess)
// (or call renderTransitionOverlay(postProcess) yourself) each frame to actually paint it.

// Timers — use TweenManager instance (create in onLoad, destroy in onDestroy)
tweens.after(2.0, fn)    // one-shot; cancelled when tweens.destroy() is called
tweens.every(0.5, fn)    // repeating; cancelled when tweens.destroy() is called

// Coroutines — async game sequences
entity.startCoroutine(function* () {
  yield waitSeconds(1.0)
  yield waitUntil(() => player.isGrounded())
  SceneManager.transition('WinScene', { effect: 'wipe' })
})

// Saves — instanced, synchronous. No schema given → default GameSaveSlot shape
// ({ id, scene, data, timestamp, playtime }); pass a Zod schema as the 2nd
// constructor arg for a custom shape — load() then already validates for you.
const save = new SaveSystem()
save.save('slot-1', { scene: 'Level2', data: { score: 4200 } })
const slot = save.load('slot-1')             // GameSaveSlot | null — never throws

// Localisation — instanced; create in onLoad
// const localisation = new LocalisationSystem()
// localisation.addTranslations('en', map); localisation.setLocale('en')
localisation.t('key.name')
localisation.t('key.score', { score: 100 })

// Tilemap
const map = TilemapSystem.load('level.esmap')
map.getLayer('Collision').enablePhysics()

// Physics events — prefer registering on the PhysicsBody you already hold:
const body = entity.requireComponent(PhysicsBody)
body.onCollisionEnter((other, contact) => {})   // contact.impactForce: number
body.onCollisionExit((other, contact) => {})
body.onSensorEnter((other) => {})
body.onSensorExit((other) => {})
body.onSensorStay((other) => {})   // fires every step while inside
// entity.onCollisionEnter()/entity.onSensorEnter() also still fire, side by side

// Tweens (TweenManager — one per scene, must call update(dt) in onUpdate)
const tweens = new TweenManager()                                         // in onLoad
tweens.to(entity.position, { x: 200 }, { duration: 0.5, ease: 'bounceOut' })
tweens.after(2.0, fn)     // scene-local one-shot timer
tweens.every(5.0, fn)     // repeating timer; stops when scene unloads
tweens.update(dt)          // in onUpdate — required

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
- **docs/** — getting started, core concepts, tutorials (this companion repo)
- **ai/api-reference.json** — machine-readable full API for programmatic agent use

The engine's own documentation uses a Unity/Unreal-style layout in the engine repository:

| Path | Contents |
|---|---|
| `docs/getting-started/` | Install, first project, first run |
| `docs/guides/` | Physics, NavMesh, multiplayer, exports, and more |
| `docs/reference/` | Every class, method, property, and type |
| `docs/tutorials/` | Complete games built from scratch |
