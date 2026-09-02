# UISystem

`UISystem` is the EmptySock built-in 2D UI layer. It manages a tree of `UIComponent` nodes and renders them onto a Canvas 2D context at the end of each frame. It is framework-agnostic and works in all engine contexts (Node.js tests, browser preview, Tauri desktop).

---

## Core concepts

| Concept | Detail |
|---------|--------|
| Components are trees | Root components are created with `UISystem.create()`; children with `comp.createChild()` |
| Anchor-based layout | Each component positions itself relative to one of nine anchor points on the canvas |
| Rendering | Call `UISystem.render(ctx, w, h)` at the end of each frame to draw all components |
| Input | Call `UISystem.update(dt, pointerX, pointerY, w, h)` each frame; call `UISystem.dispatchPointerDown(x, y, w, h)` on click |

---

## Component types

| Type | Purpose |
|------|---------|
| `panel` | Filled rectangle with optional border — use as a container |
| `button` | Panel with centred text label |
| `text` | Left-aligned text |
| `image` | Reserved placeholder; actual textures are rendered by `RenderSystem` |
| `progress-bar` | Horizontal fill bar driven by `.value` (0..1) |
| `slider` | Draggable thumb driven by `.value` (0..1) |
| `toggle` | Checkbox driven by `.checked` |

---

## Creating components

```typescript
import { UISystem } from '@emptysock/engine'

// Root component anchored to the bottom-left
const hud = UISystem.create('panel', {
  x: 10, y: -60,
  width: 200, height: 50,
  anchor: 'bottom-left',
  style: { backgroundColor: 0x1a1a2e, opacity: 0.85 },
})

// Child text label (relative to hud)
const label = hud.createChild('text', {
  x: 8, y: 12,
  text: 'Score: 0',
  style: { color: 0xffffff, fontSize: 18 },
})

// Update text each frame
label.text = `Score: ${score}`
```

---

## Anchor values

`top-left` · `top-center` · `top-right`  
`middle-left` · `middle-center` · `middle-right`  
`bottom-left` · `bottom-center` · `bottom-right`

---

## UIStyle properties

| Property | Type | Notes |
|----------|------|-------|
| `backgroundColor` | `number` | 0xRRGGBB |
| `color` | `number` | 0xRRGGBB — text or accent colour |
| `fontSize` | `number` | px |
| `fontFamily` | `string` | CSS font family |
| `borderColor` | `number` | 0xRRGGBB |
| `borderWidth` | `number` | px |
| `borderRadius` | `number` | px (visual only — hit-test is still rect) |
| `opacity` | `number` | 0..1 |
| `padding` | `number` | px (informational — layout is manual) |

---

## Animations

Built-in animations run automatically when you call the animation method. They are ticked by `UISystem.update(dt)`.

```typescript
// Fade in over 0.4 s when the panel appears
hud.fadeIn(0.4)

// Slide in from the left edge (60 px offset) over 0.3 s
hud.slideIn('left', 60, 0.3)

// Fade out and hide when dismissed
hud.fadeOut(0.25)
```

Animation types: `fadeIn(duration)` · `fadeOut(duration)` · `slideIn('left'|'right'|'up'|'down', distance, duration)`

---

## Hover style

```typescript
const btn = UISystem.create('button', {
  width: 120, height: 36,
  text: 'Play',
  style: { backgroundColor: 0x3c2d6e },
})
btn.setHoverStyle({ backgroundColor: 0x5c4da0 })
btn.onClick(() => scene.loadScene('GameScene'))
```

The hover style is merged over the base style while the pointer is over the component. It is cleared automatically on pointer-leave. Requires `UISystem.handlePointerMove()` or `UISystem.update(dt, px, py, w, h)` to receive pointer position.

---

## Frame loop integration

```typescript
class MyScene extends Scene {
  private _ctx!: CanvasRenderingContext2D
  private _w = 800
  private _h = 600

  async onLoad() {
    this._ctx = /* get your Canvas 2D context */
    const btn = UISystem.create('button', { ... })
    btn.fadeIn(0.3)
  }

  onUpdate(dt: number) {
    // Tick animations and hover
    UISystem.update(dt)

    // After game rendering, draw UI on top
    UISystem.render(this._ctx, this._w, this._h)
  }

  onDestroy() {
    UISystem.clear()
  }
}
```

---

## Input wiring

```typescript
canvas.addEventListener('click', (e) => {
  UISystem.dispatchPointerDown(e.offsetX, e.offsetY, canvas.width, canvas.height)
})
canvas.addEventListener('pointermove', (e) => {
  UISystem.handlePointerMove(e.offsetX, e.offsetY, canvas.width, canvas.height)
})
```

---

## Cleanup

```typescript
// Remove a single root component
UISystem.remove(hud)

// Clear all components (call in scene onDestroy)
UISystem.clear()
```

---

## Rules

- Never call `UISystem.create()` inside `onUpdate()` — create in `onLoad()`, update in `onUpdate()`.
- Always call `UISystem.clear()` in `onDestroy()` — components persist until explicitly removed.
- Opacity animations work by mutating `style.opacity`. Do not set `style.opacity` manually while a fade animation is active.
