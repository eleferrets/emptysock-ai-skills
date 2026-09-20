# ViewportSystem

`ViewportSystem` handles design-resolution scaling so a game authored at one fixed resolution fits any container: browser window, IDE preview pane, or a phone in either orientation. Pair it with `RenderPipeline` in every game that runs on more than one screen size — `RenderPipeline` draws, `ViewportSystem` scales what it drew to fit.

---

## Setup (in onLoad)

```typescript
import { RenderPipeline, ViewportSystem, CameraSystem } from '@emptysock/engine'

class GameScene extends Scene {
  private _render = new RenderPipeline()
  private _camera = new CameraSystem()
  private _viewport = new ViewportSystem()

  override async onLoad(): Promise<void> {
    await this._render.init({ width: 1280, height: 720 })
    document.body.appendChild(this._render.canvas)
    this._camera.attach(this._render.stage)

    this._viewport.init(
      { designWidth: 1280, designHeight: 720, scaleMode: 'fit' },
      { renderTarget: this._render, cameraSystem: this._camera },
    )
  }

  override onDestroy(): void {
    this._viewport.destroy()
    this._render.destroy()
  }
}
```

`init()` starts listening for window resize and orientation-change and immediately computes the current viewport size — there is no separate "start" call.

---

## Scale modes

| Mode | Behaviour |
|---|---|
| `'fit'` | Letterboxes — scales to the largest size that fits entirely inside the container. Never crops. |
| `'fill'` | Cover-crops — scales to the smallest size that fully covers the container. Never letterboxes. |
| `'stretch'` | Fills the container exactly, ignoring aspect ratio. |

```typescript
this._viewport.setScaleMode('fill')
this._viewport.setDesignResolution(1920, 1080)   // change the authored resolution at runtime
```

Read the current computed size (e.g. to lay out screen-anchored UI) via `this._viewport.size` — `{ width, height, offsetX, offsetY, scale }`.

For math that does not need a DOM (unit tests, tooling), call the pure function directly:

```typescript
import { computeViewportSize } from '@emptysock/engine'

const size = computeViewportSize(1280, 720, 800, 600, 'fit')
```

---

## Safe-area insets

On devices with a notch or a home indicator, read the current safe-area insets to keep UI clear of them:

```typescript
const insets = this._viewport.getSafeAreaInsets()   // { top, right, bottom, left }, all zero outside a supporting browser
hud.setPadding(insets.top, insets.right, insets.bottom, insets.left)
```

---

## GPU-tier render defaults

`gpuTierRenderDefaults()` maps a `GPUTier` (see `skills/00-quickstart.md`'s GPU tier table) to conservative `antialias`/`resolution` defaults, for feeding into `RenderSystem`/`RenderPipeline` init:

```typescript
import { gpuTierRenderDefaults } from '@emptysock/engine'

const defaults = gpuTierRenderDefaults(detectedTier, window.devicePixelRatio)
await this._render.init({ width: 1280, height: 720, ...defaults })
```

`"potato"`/`"low"` tiers disable antialiasing and cap resolution at 1. `"mid"` caps at 1.5. `"high"`/`"ultra"` cap at 2.

---

## Rules

| Wrong | Right |
|---|---|
| Reading `window.innerWidth`/`innerHeight` directly for layout | Read `viewport.size` after `init()`/`recompute()` |
| Skipping `viewport.destroy()` in `onDestroy` | Always call it — removes the resize/orientation listeners |
| Assuming safe-area insets are non-zero in tests/Node | `getSafeAreaInsets()` returns all zeros outside a browser context by design |
| Hardcoding `antialias`/`resolution` for all devices | Use `gpuTierRenderDefaults(tier)` so low-end GPUs get a usable frame rate |
