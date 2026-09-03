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
import { Scene } from '@emptysock/engine'

export class BoxScene extends Scene {
  // Private fields store data that belongs to this scene.
  // HTMLCanvasElement | null means it starts as null and becomes a canvas later.
  private _canvas: HTMLCanvasElement | null = null

  // _x tracks where the box is horizontally (in pixels from the left edge).
  private _x = 40

  // onLoad runs once, before the first frame. Use it to set up your scene.
  override async onLoad(): Promise<void> {
    // Create a canvas element and add it to the page.
    this._canvas = document.createElement('canvas')
    this._canvas.width = 800
    this._canvas.height = 600
    document.body.appendChild(this._canvas)
  }

  // onUpdate runs every frame (about 60 times per second).
  // dt is "delta time" — the number of seconds since the last frame (~0.016 at 60fps).
  override onUpdate(dt: number): void {
    // Move the box 150 pixels per second to the right.
    // Multiplying by dt makes the speed the same regardless of frame rate.
    this._x += 150 * dt

    // If the box goes past the right edge, wrap it back to the left.
    if (this._x > 760) {
      this._x = 40
    }

    // Draw the current frame.
    this._draw()
  }

  // _draw is a helper we call from onUpdate. It handles the actual drawing.
  private _draw(): void {
    // We stored _canvas as possibly null, so we check before using it.
    const canvas = this._canvas
    if (canvas === null) return

    const ctx = canvas.getContext('2d')
    if (ctx === null) return

    // Paint the background (dark navy blue).
    ctx.fillStyle = '#1a1a2e'
    ctx.fillRect(0, 0, canvas.width, canvas.height)

    // Paint the box (red-pink).
    ctx.fillStyle = '#e94560'
    ctx.fillRect(this._x, 280, 40, 40)
  }

  // onDestroy runs when this scene is unloaded.
  // Always clean up anything you created — remove the canvas from the page.
  override onDestroy(): void {
    this._canvas?.remove()
    this._canvas = null
  }
}
```

Now tell the engine to use this scene. Open `src/main.ts` and add your scene:

```typescript
import { Engine } from '@emptysock/engine'
import { BoxScene } from './scenes/BoxScene'

const engine = await Engine.create({
  scenes: { BoxScene },
  startScene: 'BoxScene',
  gameSpeed: 60,
})
```

Hit **Play** (or save the file if hot-reload is running). A red box slides across a dark background.

---

## What each part does

Let's walk through the scene line by line.

**The class definition:**
```typescript
export class BoxScene extends Scene {
```
`extends Scene` means BoxScene is a scene. The engine knows to call `onLoad`, `onUpdate`, and `onDestroy` on it automatically.

**Private fields:**
```typescript
private _canvas: HTMLCanvasElement | null = null
private _x = 40
```
`private` means these fields can only be read and changed inside this class — no other code can accidentally mess with them. The `_` prefix is a convention that flags something as private. `HTMLCanvasElement | null` means "this will be a canvas element, but starts as null until we create one in onLoad."

**The null checks:**
```typescript
const canvas = this._canvas
if (canvas === null) return
```
TypeScript insists you handle the possibility that `_canvas` is null before you use it. Copying it into a local variable (`const canvas`) and checking it lets TypeScript know that below the `if`, canvas is definitely not null. This is safer than the `!` shortcut (which would hide the problem rather than handle it).

**Delta time:**
```typescript
this._x += 150 * dt
```
`dt` is the number of seconds since the last frame. At 60fps it's roughly 0.016. Multiplying speed by dt makes the box move at 150 pixels per second regardless of whether the game is running at 30fps or 120fps. Without this, the box would move twice as fast on a 120fps monitor.

**Cleanup:**
```typescript
this._canvas?.remove()
```
The `?.` is optional chaining — it only calls `.remove()` if `_canvas` is not null. This removes the canvas element from the page when the scene unloads, so you don't end up with invisible canvases piling up in the DOM.

---

## How to add a second entity

Right now BoxScene just has a canvas and draws directly to it. To add more moving things, add more private fields and draw them in `_draw`.

Here's BoxScene with a second box that moves vertically:

```typescript
import { Scene } from '@emptysock/engine'

export class BoxScene extends Scene {
  private _canvas: HTMLCanvasElement | null = null
  private _x = 40      // red box: horizontal position
  private _y = 280     // blue box: vertical position

  override async onLoad(): Promise<void> {
    this._canvas = document.createElement('canvas')
    this._canvas.width = 800
    this._canvas.height = 600
    document.body.appendChild(this._canvas)
  }

  override onUpdate(dt: number): void {
    this._x += 150 * dt
    if (this._x > 760) this._x = 40

    this._y += 80 * dt
    if (this._y > 560) this._y = 40

    this._draw()
  }

  private _draw(): void {
    const canvas = this._canvas
    if (canvas === null) return
    const ctx = canvas.getContext('2d')
    if (ctx === null) return

    // Background
    ctx.fillStyle = '#1a1a2e'
    ctx.fillRect(0, 0, canvas.width, canvas.height)

    // Red box (moves right)
    ctx.fillStyle = '#e94560'
    ctx.fillRect(this._x, 280, 40, 40)

    // Blue box (moves down)
    ctx.fillStyle = '#4488ff'
    ctx.fillRect(400, this._y, 40, 40)
  }

  override onDestroy(): void {
    this._canvas?.remove()
    this._canvas = null
  }
}
```

The pattern is always the same: store state in private fields, update state in `onUpdate`, draw from state in `_draw`.

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
