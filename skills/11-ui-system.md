# UISystem

`UISystem` is the EmptySock 2D UI overlay. It renders `Widget` nodes on top of the PixiJS scene using Canvas 2D. `UISystem` is a module-level singleton — call its methods directly or access it as `this.uiSystem` inside any `Scene` subclass.

---

## Quick start

```typescript
import {
  UISystem,
  PanelWidget,
  LabelWidget,
  ButtonWidget,
  ProgressBarWidget,
} from '@emptysock/engine'

class HUDScene extends Scene {
  private _hp: ProgressBarWidget | null = null
  private _scoreLabel: LabelWidget | null = null

  override onLoad(): void {
    this._hp = new ProgressBarWidget({
      anchor: 'top-left',
      x: 16, y: 16,
      width: 200, height: 14,
      fillColor: 0xe74c3c,
      trackColor: 0x333333,
      value: 1.0,
    })
    UISystem.add(this._hp)

    this._scoreLabel = new LabelWidget({
      anchor: 'top-right',
      x: 16, y: 16,
      text: 'Score: 0',
      fontSize: 20,
      color: 0xffffff,
    })
    UISystem.add(this._scoreLabel)
  }

  override onUpdate(dt: number): void {
    UISystem.update(dt)
  }

  override onDestroy(): void {
    UISystem.clear()
  }
}
```

---

## Widget classes

| Widget | Key properties |
|--------|---------------|
| `PanelWidget` | `background`, `border?`, `borderWidth`, `cornerRadius`, `children` |
| `LabelWidget` | `text`, `font`, `fontSize`, `color`, `align` |
| `ButtonWidget` | `label`, `icon?`, `disabled`, `animateOnHover` |
| `ImageWidget` | `src`, `scaleMode` (`stretch` / `fit` / `fill` / `none`), `tint?` |
| `ProgressBarWidget` | `value`, `min`, `max`, `fillColor`, `trackColor`, `direction` (`h`/`v`) |
| `SliderWidget` | `value`, `min`, `max`, `step`, `trackColor`, `thumbColor`, `onChange?` |
| `CheckboxWidget` | `checked`, `label`, `color`, `borderColor`, `onChange?` |

---

## Base Widget properties

All widgets share the `Widget` base class:

```typescript
widget.x        // pixel offset from anchor point
widget.y
widget.width
widget.height
widget.anchor   // 'top-left' | 'top' | 'top-right' | 'left' | 'center' | 'right' | 'bottom-left' | 'bottom' | 'bottom-right'
widget.visible  // hide without removing
widget.alpha
widget.children // Widget[] — mutable; push children onto PanelWidget for compound layouts
widget.on(event, handler)
widget.off(event, handler)
widget.animate(name, opts?)
```

**Events:** `'click'`, `'hover'`, `'hoverOut'`, `'change'`, `'animEnd'`

**Animations:** `'fadeIn'`, `'fadeOut'`, `'slideIn'`, `'slideOut'`, `'pop'`, `'shake'`  
All accept `{ duration?: number, easing?: string, direction?: 'left'|'right'|'up'|'down' }`.

---

## Buttons and events

```typescript
import { UISystem, ButtonWidget, SceneManager } from '@emptysock/engine'

const btn = new ButtonWidget({
  label: 'Retry',
  anchor: 'center',
  width: 120,
  height: 40,
})
btn.on('click', () => SceneManager.load('GameScene'))
btn.animate('fadeIn', { duration: 0.3 })
UISystem.add(btn)
```

---

## Compound panels

Build complex layouts by pushing children onto a `PanelWidget`. Children position relative to the panel's origin:

```typescript
import { UISystem, PanelWidget, LabelWidget, ButtonWidget } from '@emptysock/engine'

const panel = new PanelWidget({
  anchor: 'center',
  width: 300, height: 200,
  background: 0x1a1a2e,
})

const title = new LabelWidget({ text: 'Game Over', fontSize: 24, anchor: 'top', y: 16 })
const score = new LabelWidget({ text: 'Score: 0', fontSize: 18, anchor: 'center' })
const retry = new ButtonWidget({ label: 'Retry', anchor: 'bottom', y: 20, width: 100, height: 36 })

retry.on('click', () => SceneManager.load('GameScene'))
panel.children.push(title, score, retry)
UISystem.add(panel)
panel.animate('fadeIn')
```

---

## UISystem API

| Method | Description |
|--------|-------------|
| `UISystem.add(widget)` | Add a root widget to the overlay |
| `UISystem.removeWidget(widget)` | Remove a specific root widget |
| `UISystem.clear()` | Remove all widgets and legacy UIComponents |
| `UISystem.update(dt, px?, py?, cw?, ch?)` | Tick animations and hover state each frame |
| `UISystem.render(ctx, cw, ch)` | Draw all widgets to a Canvas 2D context |
| `UISystem.setImageLoader(loader)` | Override the default fetch-based image loader |

---

## Frame loop integration

```typescript
class MyScene extends Scene {
  override onLoad(): void {
    const btn = new ButtonWidget({ label: 'Play', anchor: 'center' })
    btn.animate('fadeIn', { duration: 0.3 })
    UISystem.add(btn)
  }

  override onUpdate(dt: number): void {
    UISystem.update(dt)
    // UISystem.render() is called automatically after PixiJS renders.
    // Only call it manually if you manage a raw Canvas 2D context yourself.
  }

  override onDestroy(): void {
    UISystem.clear()
  }
}
```

---

## Rules

- Never call `UISystem.add()` inside `onUpdate()` — create widgets in `onLoad()`, update properties in `onUpdate()`.
- Always call `UISystem.clear()` in `onDestroy()` — widgets persist until explicitly removed.
- Use `widget.visible = false` to hide temporarily; `UISystem.removeWidget(w)` to fully remove.
- `widget.children` is a plain mutable array — push onto a `PanelWidget` to nest widgets.
- Never use definite-assignment `!` on widget fields; use `T | null = null` and check before use.
