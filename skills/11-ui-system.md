# UISystem

**Use this when** you're building screen-space UI (HUDs, menus, dialogue boxes) as opposed to in-world objects. UI is entity-based: each widget is an entity with a `LayoutStyle` and `Layout` component plus one or more widget-kind components (`Label`, `PanelStyle`, `ButtonState`, `Checkbox`, `Slider`, `Progress`, `ImageWidget`). `WidgetTree` owns the parent/child tree and flexbox layout (Yoga WASM); `UISystem` hit-tests, dispatches pointer input, and draws to a Canvas 2D-style context.

---

## Quick start

```typescript
import {
  defineScene, WidgetTree, UISystem, LayoutStyle, PanelStyle, Label, Progress,
} from '@emptysock/engine'

const tree = new WidgetTree()
const ui = new UISystem(tree)          // options: { imageLoader?, fonts? }

const HudScene = defineScene({
  async onLoad(scene) {
    await tree.init()                  // loads Yoga WASM; must finish before layout()

    const root = tree.createWidget(scene)
    const rootStyle = root.get(LayoutStyle)
    if (rootStyle !== undefined) { rootStyle.width = 1280; rootStyle.height = 720; rootStyle.padding = 16 }

    const hp = tree.createWidget(scene, root)           // child of root
    const hpStyle = hp.get(LayoutStyle)
    if (hpStyle !== undefined) { hpStyle.width = 200; hpStyle.height = 14 }
    hp.add(Progress, { value: 1, fillColor: '#e74c3c', trackColor: '#333333' })

    const score = tree.createWidget(scene, root)
    score.add(Label, { text: 'Score: 0', fontSize: 20 })
  },
  onUpdate(dt) {
    // each frame: tree.layout(scene, 1280, 720), then ui.render(scene, ctx)
  },
})
```

`tree.createWidget(scene, parent?)` spawns an entity with `LayoutStyle` + `Layout` already attached. Add widget-kind components on top with `entity.add(...)`.

---

## Widget-kind components

| Component | Fields (defaults in parentheses) |
|-----------|----------------------------------|
| `WidgetAppearance` | `visible` (true), `alpha` (1) |
| `Label` | `text`, `color` ('#ffffff'), `fontSize` (14), `font` ('sans-serif'), `fontId` (''), `align` (0 left, 1 center, 2 right) |
| `PanelStyle` | `background` ('#1a1a2e'), `borderColor`, `borderWidth`, `borderRadius` |
| `ButtonState` | `label`, `color`, `background`, `hoverBackground`, `pressedBackground`, `borderRadius`, `fontSize`, `font`, `fontId`, `disabled`, `state` (0 idle, 1 hover, 2 pressed; set by UISystem) |
| `Checkbox` | `checked`, `label`, `color`, `background`, `borderColor`, `fontSize`, `font`, `fontId` |
| `Slider` | `value`, `min` (0), `max` (1), `step` (0 = continuous), `trackColor`, `thumbColor` |
| `Progress` | `value`, `min`, `max`, `trackColor`, `fillColor` |
| `ImageWidget` | `src` (texture path, loaded through `TextureStore`) |

`LayoutStyle` fields: `flexDirection` (0 column, 1 row), `width`, `height` (-1 = auto), `flexGrow`, `flexShrink`, `padding`, `gap`, `positionType` (0 relative, 1 absolute), `left`, `top`, `overflow` (0 visible, 1 hidden, 2 scroll), `scrollX`, `scrollY`. `Layout` (`x`, `y`, `width`, `height`) is the computed result written by `tree.layout(...)`.

`resolveAnchoredPosition(anchor, x, y, width, height, containerWidth, containerHeight)` returns `{ left, top }` for a nine-point `WidgetAnchor` (`'top-left'` ... `'bottom-right'`); assign the result to an absolute-positioned widget's `LayoutStyle.left/top`.

---

## Pointer input and clicks

`UISystem` has no event emitter. Feed it pointer events and read component state:

| Method | Description |
|--------|-------------|
| `ui.hitTest(scene, x, y)` | Topmost visible widget entity under the point, or undefined |
| `ui.dispatchPointerDown(scene, x, y, pointerId?)` | Begin a press (buttons go to `state = 2`, sliders update) |
| `ui.dispatchPointerDrag(scene, x, y, pointerId?)` | Update a press; past 6px it becomes a drag (no click) |
| `ui.dispatchPointerUp(scene, x, y, pointerId?)` | Complete a press; returns true if a click fired (toggles a `Checkbox`) |
| `ui.cancelPointer(pointerId?)` | Abort a press without a click |
| `ui.updateHover(scene, x, y)` | Set `ButtonState.state` to hover/idle |
| `ui.render(scene, ctx)` | Draw all visible widgets; `ctx` is an `IUIRenderer` (a `CanvasRenderingContext2D` satisfies it) |

To react to a button click, use the boolean from `dispatchPointerUp` together with `ui.hitTest` (or the entity returned by `dispatchPointerDown`) and check which entity has `ButtonState`.

```typescript
const pressed = ui.dispatchPointerDown(scene, x, y)
// ...later, on release:
const clicked = ui.dispatchPointerUp(scene, x, y)
if (clicked && pressed !== undefined && pressed.has(ButtonState)) startGame()
```

---

## WidgetTree API

| Method | Description |
|--------|-------------|
| `await tree.init()` | Load Yoga WASM (check `tree.ready`) |
| `tree.createWidget(scene, parent?)` | Spawn a widget entity under an optional parent |
| `tree.destroyWidget(scene, entity)` | Destroy one widget (does not cascade to children; destroy them first) |
| `tree.parentOf(scene, entity)` | The parent widget or undefined |
| `tree.orderedWidgets(scene)` | Root-first draw order |
| `tree.layout(scene, rootWidth, rootHeight)` | Compute `Layout` for every widget (throws before `init()`) |
| `tree.destroy()` | Free Yoga nodes |

---

## Rules

- Await `tree.init()` before the first `tree.layout()` call.
- Build widgets once in `onLoad`; mutate component fields afterwards. Never create widgets in `onUpdate`.
- Destroy children before their parent (`tree.destroyWidget` does not cascade), and call `tree.destroy()` when the scene unloads.
- Keep interactive widgets at least 44px square for touch (guideline only; nothing enforces it).
- Skip the `!` on `entity.get(...)`; it returns `T | undefined`, so check before use.
