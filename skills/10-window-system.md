# WindowSystem — window management

**Use this when** you need to control the game window: fullscreen toggles, resizing, title changes, that kind of thing. `windowSystem` wraps Tauri's native window API with a transparent browser fallback, so your game code never has to know or care which one it's running under.

---

## Compile-time constants

These identifiers are replaced with literal values at build time. No import needed — they are globals.

| Constant | Type | Source |
|---|---|---|
| `PROJECT_TITLE` | `string` | Project → window title |
| `PROJECT_NAME` | `string` | Project name |
| `GAME_WIDTH` | `number` | Project → window width |
| `GAME_HEIGHT` | `number` | Project → window height |
| `DEBUG` | `boolean` | Build mode (debug = `true`) |

```typescript
// Use in any game file without imports:
windowSystem.setTitle(`${PROJECT_TITLE} — Wave ${this.wave}`)
// GAME_WIDTH and GAME_HEIGHT hold the configured canvas dimensions:
await windowSystem.setSize(GAME_WIDTH, GAME_HEIGHT)
if (DEBUG) console.log('dev build')
```

---

## Apply project settings on startup

Call `apply()` once in your root scene's `onLoad`. It reads any subset of `WindowConfig`.

```typescript
import { windowSystem } from '@emptysock/engine'

override async onLoad(): Promise<void> {
  await windowSystem.apply({
    mode:      'windowed',
    width:     GAME_WIDTH,
    height:    GAME_HEIGHT,
    title:     PROJECT_TITLE,
    resizable: true,
    minWidth:  640,
    minHeight: 360,
  })
}
```

---

## Window modes

| Mode | Desktop (Tauri) | Browser |
|---|---|---|
| `'windowed'` | Decorated window at configured size | Canvas at configured size |
| `'fullscreen'` | Exclusive fullscreen | `requestFullscreen()` |
| `'borderless'` | Undecorated maximised window | Canvas fills viewport |

```typescript
// Toggle fullscreen (e.g. on F key — assuming this.input is an attached InputSystem):
if (this.input.isKeyPressed('KeyF')) {
  const next = windowSystem.currentMode === 'fullscreen' ? 'windowed' : 'fullscreen'
  await windowSystem.setMode(next)
}
```

F11 toggles native fullscreen in the browser automatically — nothing for you to wire up.

---

## Runtime API

```typescript
import { windowSystem, type WindowMode } from '@emptysock/engine'

await windowSystem.setMode(mode: WindowMode): Promise<void>
await windowSystem.setSize(width: number, height: number): Promise<void>
await windowSystem.setTitle(title: string): Promise<void>
await windowSystem.setResizable(resizable: boolean): Promise<void>
await windowSystem.setMinSize(width: number, height: number): Promise<void>
await windowSystem.setPosition(x: number, y: number): Promise<void>
await windowSystem.center(): Promise<void>
await windowSystem.setAlwaysOnTop(value: boolean): Promise<void>

windowSystem.currentMode   // WindowMode
windowSystem.currentConfig // Readonly<WindowConfig>
```

---

## WindowConfig shape

```typescript
interface WindowConfig {
  mode:      'windowed' | 'fullscreen' | 'borderless'
  width:     number
  height:    number
  title:     string
  resizable: boolean
  minWidth:  number
  minHeight: number
}
```

---

## Performance / resolution scaling

Use `setSize` for dynamic resolution changes (e.g. quality settings menu):

```typescript
const resolutions: Record<string, [number, number]> = {
  low:    [1280,  720],
  medium: [1920, 1080],
  high:   [2560, 1440],
}

async function applyQuality(preset: keyof typeof resolutions): Promise<void> {
  const [w, h] = resolutions[preset]
  await windowSystem.setSize(w, h)
}
```

---

## Top-level async/await

Game scripts can `await` at module level — the build pipeline wraps the bundle so this just works.

```typescript
// game.ts — top-level await is fine
import { Engine, windowSystem } from '@emptysock/engine'
import { GameScene } from './GameScene'

await windowSystem.apply({ mode: 'windowed', width: GAME_WIDTH, height: GAME_HEIGHT, title: PROJECT_TITLE })

const engine = await Engine.create({ scenes: { GameScene }, startScene: 'GameScene' })
await engine.start()
```
