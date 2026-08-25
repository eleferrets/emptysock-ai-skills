# Getting Started with EmptySock

This guide gets you from zero to a running game in under 10 minutes.

---

## Prerequisites

- **Node.js** 20+ and **pnpm** 9+
- **Rust** toolchain (for Tauri — `curl https://sh.rustup.rs | sh`)
- For Android export: Android Studio + SDK
- For iOS export: macOS + Xcode
- For Raspberry Pi export: no extra tooling needed

---

## Install EmptySock

Download and run the installer for your platform from the EmptySock releases page.

The IDE installs as a native application. No terminal setup required to use the IDE.

---

## Create your first project

1. Open EmptySock.
2. Click **New Project**.
3. Choose a name and a template. For a first project, **Platformer** is recommended —
   it ships with a working character, level, and controls already wired up.
4. Choose your target platforms. You can always add more later.
5. Click **Create**. The project opens immediately.
6. Hit **Play** in the toolbar. The game runs.

That's it. You have a running game.

---

## Project layout

```
my-game/
  emptysock.project.json   ← project settings (name, scenes, targets, FPS)
  src/                     ← TypeScript source — your game code lives here
    main.ts                ← engine boot file
    scenes/                ← one file per scene
    entities/              ← entity factory functions or classes
    components/            ← custom components
  assets/                  ← game assets — never put source code here
    sprites/               ← PNG, WebP image files
    audio/                 ← MP3, OGG, WAV files
    tilemaps/              ← .esmap tilemap files
    i18n/                  ← translation JSON files (en.json, fr.json, etc.)
  meta/                    ← export metadata — not bundled into game
    icon/
      icon-1024.png        ← your master app icon (provide this, rest is auto-generated)
    splash/
      splash.png           ← optional splash image (disabled by default)
  export/                  ← compiled game outputs (auto-generated, gitignored)
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
- Dialogue tree editor (for visual novels)

**Right panel** — entity inspector. Select an entity in the canvas to edit its components.

**Bottom panel** — console output, audio mixer, performance profiler.

All panels are resizable. Drag panel edges to adjust.

---

## The game loop

EmptySock uses a simple fixed-FPS game loop. Your code goes in three places:

```typescript
override async onLoad(): Promise<void> {
  // Runs once before the scene starts. Load assets, create entities.
  // This is async — you can await asset loads here.
}

override onUpdate(dt: number): void {
  // Runs every frame. dt = seconds since last frame (~0.0167 at 60fps).
  // Move things, check input, update logic.
  // Never use async/await here — use Timer or coroutines.
}

override onDestroy(): void {
  // Runs when the scene unloads. Cancel timers, clean up.
}
```

---

## Your first custom scene

1. In the left sidebar, right-click `src/scenes/` → **New file** → `MyScene.ts`.
2. Write:

```typescript
import { Scene, type SceneConfig, Sprite, Input } from '@emptysock/engine'

export class MyScene extends Scene {
  static readonly config: SceneConfig = {
    renderMode: '2d',
    gameSpeed: 60,
  }

  override async onLoad(): Promise<void> {
    const box = scene.createEntity('Box')
    box.addComponent(Sprite, { texture: 'crate.png', anchor: { x: 0.5, y: 0.5 } })
    box.setPosition(400, 300)
  }

  override onUpdate(dt: number): void {
    const box = scene.getEntity('Box')
    if (box && Input.isDown('ArrowRight')) {
      box.translate(200 * dt, 0)
    }
  }

  override onDestroy(): void {}
}
```

3. Register it in `src/main.ts`:

```typescript
import { MyScene } from './scenes/MyScene'

const engine = await Engine.create({
  scenes: { MyScene, ...otherScenes },
  startScene: 'MyScene',
  gameSpeed: 60,
})
```

4. Hit **Play**. Your box moves with the arrow key.

---

## Using an external code editor

EmptySock has a built-in Monaco editor. To use VS Code or another editor instead:

1. Open your project in EmptySock.
2. In the toolbar, click **Edit in VS Code** (or your configured editor).
3. Both editors can be open at the same time. EmptySock watches for file changes
   and hot-reloads the preview automatically on save.

---

## Adding assets

Drag any supported file into the asset browser panel:

- **Images:** PNG, JPEG, WebP, GIF, SVG
- **Audio:** MP3, OGG, WAV, WebM, FLAC
- **Fonts:** TTF, OTF, WOFF2
- **Data:** JSON, CSV
- **3D models:** glTF (.gltf, .glb), USDZ

EmptySock copies the file into your `assets/` folder and creates an asset descriptor.
Reference assets by filename in your code: `{ texture: 'hero.png' }`.

---

## Exporting your game

1. Click **Export** in the toolbar.
2. Select your target platforms.
3. Click **Export All**.

Output is placed in `export/` in your project folder.

| Platform | Output |
|---|---|
| Web | `export/web/` folder + optional `.zip` |
| Windows | `.exe` installer + portable `.zip` |
| macOS | `.app` bundle + `.dmg` installer |
| Linux | `.AppImage` + `.deb` package |
| Android | `.apk` (direct install) + `.aab` (Play Store) |
| iOS | `.ipa` (TestFlight / App Store) |
| Raspberry Pi | `.tar.gz` with `install.sh` |

All exports: no source code, no source maps, fully minified.

---

## Next steps

- Read `skills/00-quickstart.md` for the most common code patterns
- Read `docs/core-concepts.md` to understand the ECS model
- Browse `skills/` for the system you're working with
- For AI assistance, drop `ai/CLAUDE.md` into your project root
