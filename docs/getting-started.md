# Getting Started with EmptySock

Welcome! If you've never made a game before, you're in the right place.

---

## What is EmptySock?

EmptySock is a game engine — a toolkit that handles the boring parts of making games (drawing to the screen, playing sounds, tracking time, reading keyboard input) so you can focus on the fun parts (your game's actual logic and ideas). You write code in TypeScript, hit Play, and your game runs in the browser or as a desktop app. That's it.

---

## Do I need to know TypeScript?

Not really, no. If you've written JavaScript before — even just DOM stuff like `document.querySelector` or a few `addEventListener` calls — you know enough to start. TypeScript is basically JavaScript with labels that tell you what type of thing a variable holds. The labels help catch mistakes early. You'll pick up the TypeScript parts as you go, and the examples in this guide are fully annotated so you can copy the patterns even if you don't understand every detail yet.

The one thing to know upfront: TypeScript uses `: TypeName` after a variable to say what kind of value it holds.

```typescript
let name: string = 'Ada'   // name holds text
let speed: number = 200    // speed holds a number
let active: boolean = true // active holds true/false
```

That's most of what you need to start.

---

## Prerequisites

You need:

- **Node.js 20+** — the JavaScript runtime. Download from [nodejs.org](https://nodejs.org).
- **pnpm 9+** — a package manager. Once Node is installed, run `npm install -g pnpm`.

That's all you need to run EmptySock and build web games.

Optional extras:
- **Rust** — only needed if you want to build a desktop app (.exe, .app, .AppImage). Install with `curl https://sh.rustup.rs | sh`.
- **Android Studio** — only for Android export.
- **Xcode on macOS** — only for iOS export.

If you're just starting out, skip the optional stuff and run in the browser. You can always add them later.

---

## Install and run

Three commands and you're up:

```bash
git clone https://github.com/emptysock/emptysock-engine.git my-game
cd my-game
pnpm install && pnpm dev
```

The IDE opens in your browser at `http://localhost:5173`. Click **Play** and the demo scene runs.

---

## Your first game — a moving colored box

Before we touch the IDE, let's write code that actually does something you can see. This example draws a red box on screen and moves it across from left to right. No image files required — just code.

Create a new file at `src/scenes/BoxScene.ts` and paste this in:

```typescript
import { defineScene } from '@emptysock/engine'

// Private-ish state for this scene — just plain variables in the module's
// closure, since a scene here is a plain object, not a class you subclass.
let canvas: HTMLCanvasElement | null = null
let x = 40   // horizontal position of the box, in pixels from the left edge

export const BoxScene = defineScene({
  // onLoad runs once, before the first frame. Use it to set up your scene.
  async onLoad() {
    canvas = document.createElement('canvas')
    canvas.width = 800
    canvas.height = 600
    document.body.appendChild(canvas)
  },

  // onUpdate runs every frame (about 60 times per second).
  // dt is "delta time" — the number of seconds since the last frame (~0.016 at 60fps).
  // It cannot be async — that's enforced by the type checker, not just a style rule.
  onUpdate(dt: number) {
    // Move the box 150 pixels per second to the right.
    // Multiplying by dt makes the speed the same regardless of frame rate.
    x += 150 * dt

    // If the box goes past the right edge, wrap it back to the left.
    if (x > 760) x = 40

    draw()
  },

  // onUnload runs when this scene is unloaded.
  // Always clean up anything you created — remove the canvas from the page.
  onUnload() {
    canvas?.remove()
    canvas = null
  },
})

// A plain helper function we call from onUpdate. It handles the actual drawing.
function draw(): void {
  // canvas might still be null if onLoad hasn't finished — check before using it.
  if (canvas === null) return

  const ctx = canvas.getContext('2d')
  if (ctx === null) return

  // Paint the background (dark navy blue).
  ctx.fillStyle = '#1a1a2e'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  // Paint the box (red-pink).
  ctx.fillStyle = '#e94560'
  ctx.fillRect(x, 280, 40, 40)
}
```

Now tell the engine to use this scene. Open `src/main.ts` and add your scene:

```typescript
import { Game } from '@emptysock/engine'
import { BoxScene } from './scenes/BoxScene'

const game = new Game()
await game.loadScene(BoxScene)

// Your host's render loop calls this every frame:
function tick(dt: number): void {
  game.update(dt)
  requestAnimationFrame(() => tick(/* time since last tick, in seconds */ 1 / 60))
}
tick(1 / 60)
```

Hit **Play** (or save the file if hot-reload is running). A red box slides across a dark background.

---

## What each part does

Let's walk through the scene line by line.

**The scene definition:**
```typescript
export const BoxScene = defineScene({ onLoad, onUpdate, onUnload })
```
A scene here is a plain object with a few optional hooks, not a class you subclass — `defineScene` just wraps it so TypeScript can catch a mistake like an accidentally-async `onUpdate` at compile time. `Game.loadScene(BoxScene)` calls `onLoad`, then `onUpdate` every frame, then `onUnload` when the scene is torn down.

**The null check:**
```typescript
if (canvas === null) return
```
TypeScript insists you handle the possibility that `canvas` is null before you use it — it starts `null` until `onLoad` actually creates one. This is safer than the `!` shortcut (which would hide the problem rather than handle it).

**Delta time:**
```typescript
x += 150 * dt
```
`dt` is the number of seconds since the last frame. At 60fps it's roughly 0.016. Multiplying speed by dt makes the box move at 150 pixels per second regardless of whether the game is running at 30fps or 120fps. Without this, the box would move twice as fast on a 120fps monitor.

**Cleanup:**
```typescript
canvas?.remove()
```
The `?.` is optional chaining — it only calls `.remove()` if `canvas` is not null. This removes the canvas element from the page when the scene unloads, so you don't end up with invisible canvases piling up in the DOM.

---

## How to add a second entity

Right now BoxScene just has a canvas and draws directly to it. To add more moving things, add more private fields and draw them in `_draw`.

Here's BoxScene with a second box that moves vertically:

```typescript
import { defineScene } from '@emptysock/engine'

let canvas: HTMLCanvasElement | null = null
let x = 40      // red box: horizontal position
let y = 280     // blue box: vertical position

export const BoxScene = defineScene({
  async onLoad() {
    canvas = document.createElement('canvas')
    canvas.width = 800
    canvas.height = 600
    document.body.appendChild(canvas)
  },

  onUpdate(dt: number) {
    x += 150 * dt
    if (x > 760) x = 40

    y += 80 * dt
    if (y > 560) y = 40

    draw()
  },

  onUnload() {
    canvas?.remove()
    canvas = null
  },
})

function draw(): void {
  if (canvas === null) return
  const ctx = canvas.getContext('2d')
  if (ctx === null) return

  // Background
  ctx.fillStyle = '#1a1a2e'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  // Red box (moves right)
  ctx.fillStyle = '#e94560'
  ctx.fillRect(x, 280, 40, 40)

  // Blue box (moves down)
  ctx.fillStyle = '#4488ff'
  ctx.fillRect(400, y, 40, 40)
}
```

The pattern is always the same: store state in module-level variables (or, once you're past canvas experiments, in real ECS components), update state in `onUpdate`, draw from state in `draw`.

---

## Project layout

Once you start adding files, here's where everything lives:

```
my-game/
  emptysock.project.json   ← project settings (name, scenes, FPS)
  src/
    main.ts                ← engine boot — registers your scenes here
    scenes/                ← one file per scene (BoxScene.ts, MenuScene.ts, etc.)
    entities/              ← optional: factory functions for complex entities
    components/            ← optional: your own custom components
  assets/
    sprites/               ← PNG, WebP images
    audio/                 ← MP3, OGG, WAV files
    tilemaps/              ← .esmap tilemap files
    i18n/                  ← translation files (en.json, fr.json, etc.)
  export/                  ← built game outputs (auto-generated, gitignored)
```

---

## Understanding the editor

The IDE has five main areas:

**Left sidebar** — project tree, asset browser, tool palette.

**Centre workspace** — tabs that switch between:
- Code editor (Monaco, full TypeScript with autocomplete)
- Canvas preview (the live game)
- Tilemap editor
- Particle editor
- Dialogue tree editor

**Right panel** — entity inspector.

**Bottom panel** — console output, audio mixer, performance profiler.

All panels are resizable. Drag panel edges to adjust.

---

## Adding assets

Drag any supported file into the asset browser panel:

- **Images:** PNG, JPEG, WebP, GIF, SVG
- **Audio:** MP3, OGG, WAV, WebM, FLAC
- **Fonts:** TTF, OTF, WOFF2
- **Data:** JSON, CSV
- **3D models:** glTF (.gltf, .glb)

EmptySock copies the file into `assets/` and creates an asset descriptor. Reference assets by filename in your code: `{ texture: 'hero.png' }`.

---

## Exporting your game

1. Click **Export** in the toolbar.
2. Select your target platforms.
3. Click **Export All**.

Output lands in `export/` inside your project folder.

| Platform | Output |
|---|---|
| Web | `export/web/` folder + optional `.zip` |
| Windows | `.exe` installer + portable `.zip` |
| macOS | `.app` bundle + `.dmg` installer |
| Linux | `.AppImage` + `.deb` package |
| Android | `.apk` + `.aab` (Play Store) |
| iOS | `.ipa` (TestFlight / App Store) |

All exports are minified — no source code ships to players.

---

## Next steps

You have a moving box. Here's where to go from here:

- **`docs/core-concepts.md`** — understand the ECS model, scenes, physics, input, and coroutines
- **`skills/00-quickstart.md`** — the most common code patterns on one page
- **`skills/`** — topic files for physics, audio, saves, NavMesh, and more
- **`ai/CLAUDE.md`** — drop this into your project root so an AI assistant understands the engine rules

---

## Engine documentation

The engine docs use a Unity/Unreal-style layout. Everything below lives in the engine
repository alongside the source:

| Section | Path in engine repo | What's there |
|---|---|---|
| Install and first project | `docs/getting-started/` | Node/Rust/platform SDKs, install, first run |
| How-to guides | `docs/guides/` | Physics, NavMesh, multiplayer, exports, and more |
| Full API reference | `docs/reference/` | Every class, method, property, and type |
| Step-by-step tutorials | `docs/tutorials/` | Complete games built from scratch |
