# ViewportSystem

**Use this when** your game is authored at one fixed resolution but needs to fit whatever container it lands in — a browser window, the IDE preview pane, a phone flipped sideways. `ViewportSystem` handles that scaling. Pair it with `RenderPipeline` for any game that runs on more than one screen size: `RenderPipeline` draws, `ViewportSystem` scales what it drew to fit.

---

## Setup

```typescript
import { RenderPipeline, ViewportSystem, CameraSystem } from '@emptysock/engine'

const render = new RenderPipeline()
const camera = new CameraSystem()
const viewport = new ViewportSystem()   // each scene's lifecycle context also provides one as ctx.viewport

await render.init({ width: 1280, height: 720 })
document.body.appendChild(render.canvas)
camera.attach(render.stage)

viewport.init(
  { designWidth: 1280, designHeight: 720, scaleMode: 'fit' },   // optional: container?: HTMLElement
  { renderTarget: render, cameraSystem: camera },
)

// on shutdown:
viewport.destroy()
render.destroy()
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
viewport.setScaleMode('fill')
viewport.setDesignResolution(1920, 1080)   // change the authored resolution at runtime
```

Read the current computed size (e.g. to lay out screen-anchored UI) via `viewport.size` — `{ width, height, offsetX, offsetY, scale }`.

For math that does not need a DOM (unit tests, tooling), call the pure function directly:

```typescript
import { computeViewportSize } from '@emptysock/engine'

const size = computeViewportSize(1280, 720, 800, 600, 'fit')
```

---

## Safe-area insets

On devices with a notch or a home indicator, read the current safe-area insets to keep UI clear of them:

```typescript
const insets = viewport.getSafeAreaInsets()   // { top, right, bottom, left }, all zero outside a supporting browser
applyHudPadding(insets.top, insets.right, insets.bottom, insets.left)   // your own layout code
```

---

## GPU-tier render defaults

`gpuTierRenderDefaults()` maps a GPU tier (`'potato' | 'low' | 'mid' | 'high' | 'ultra'`) to conservative `antialias`/`resolution` defaults, for feeding into `RenderPipeline` init:

```typescript
import { gpuTierRenderDefaults } from '@emptysock/engine'

const defaults = gpuTierRenderDefaults(detectedTier, window.devicePixelRatio)
await render.init({ width: 1280, height: 720, ...defaults })
```

`"potato"`/`"low"` tiers disable antialiasing and cap resolution at 1. `"mid"` caps at 1.5. `"high"`/`"ultra"` cap at 2.

---

## Rules

| Wrong | Right |
|---|---|
| Reading `window.innerWidth`/`innerHeight` directly for layout | Read `viewport.size` after `init()`/`recompute()` instead |
| Skipping `viewport.destroy()` on shutdown | Always call it; it removes the resize/orientation listeners |
| Assuming safe-area insets are non-zero in tests/Node | `getSafeAreaInsets()` returns all zeros outside a real browser, by design |
| Hardcoding `antialias`/`resolution` for every device | Use `gpuTierRenderDefaults(tier)` so low-end GPUs still get a playable frame rate |
