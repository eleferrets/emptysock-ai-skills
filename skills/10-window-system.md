# WindowSystem: window management

**Use this when** you need to control the game window: fullscreen toggles, resizing, title changes, that kind of thing. `WindowSystem` wraps Tauri's native window API with a transparent browser fallback (each scene's lifecycle context provides one as `ctx.window`; you can also `new WindowSystem()`), so your game code never has to know or care which one it's running under.

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
await ctx.window.setTitle(`${PROJECT_TITLE} - Wave ${wave}`)
// GAME_WIDTH and GAME_HEIGHT hold the configured canvas dimensions:
await ctx.window.setSize(GAME_WIDTH, GAME_HEIGHT)
if (DEBUG) console.log('dev build')
```

---

## Apply project settings on startup

Call `apply()` once in your root scene's `onLoad`. It reads any subset of `WindowConfig`.

```typescript
import { defineScene } from '@emptysock/engine'

export const Boot = defineScene({
  async onLoad(scene, ctx) {
    await ctx.window.apply({
      mode:      'windowed',
      width:     GAME_WIDTH,
      height:    GAME_HEIGHT,
      title:     PROJECT_TITLE,
      resizable: true,
      minWidth:  640,
      minHeight: 360,
    })
  },
})
```

---

## Window modes

| Mode | Desktop (Tauri) | Browser |
|---|---|---|
| `'windowed'` | Decorated window at configured size | Canvas at configured size |
| `'fullscreen'` | Exclusive fullscreen | `requestFullscreen()` |
| `'borderless'` | Undecorated maximised window | Canvas fills viewport |

```typescript
// Toggle fullscreen on an action (e.g. bound to the F key); do this from a UI handler or coroutine,
// since onUpdate cannot be async:
if (ctx.input.wasPressed('toggleFullscreen')) {
  const next = ctx.window.currentMode === 'fullscreen' ? 'windowed' : 'fullscreen'
  void ctx.window.setMode(next)
}
```

F11 toggles native fullscreen in the browser automatically — nothing for you to wire up.

---

## Runtime API

```typescript
import { WindowSystem, type WindowMode } from '@emptysock/engine'

// `win` is a WindowSystem (ctx.window inside a scene)
await win.apply(config: Partial<WindowConfig>): Promise<void>
await win.setMode(mode: WindowMode): Promise<void>
await win.setSize(width: number, height: number): Promise<void>
await win.setTitle(title: string): Promise<void>
await win.setResizable(resizable: boolean): Promise<void>
await win.setMinSize(width: number, height: number): Promise<void>
await win.setPosition(x: number, y: number): Promise<void>
await win.center(): Promise<void>
await win.setAlwaysOnTop(value: boolean): Promise<void>

win.getSize()       // { width, height } last-known size
win.currentMode     // WindowMode
win.currentConfig   // Readonly<WindowConfig>
win.destroy()
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
  await ctx.window.setSize(w, h)
}
```

---
