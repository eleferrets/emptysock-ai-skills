# UISystem

`UISystem` is the EmptySock 2D UI overlay. It renders `Widget` nodes on top of the PixiJS scene using Canvas 2D. Each `Scene` owns a `UISystem` instance at `scene.ui` — there is no global singleton. Widgets added to `scene.ui` are automatically cleared when the scene is destroyed.

---

## Quick start

```typescript
import {
  Scene,
  PanelWidget,
  LabelWidget,
  ButtonWidget,
  ProgressBarWidget,
} from '@emptysock/engine'

class HUDScene extends Scene {
  private _hp: ProgressBarWidget | null = null
  private _scoreLabel: LabelWidget | null = null

  override async onLoad(): Promise<void> {
    this._hp = new ProgressBarWidget({
      anchor: 'top-left',
      x: 16, y: 16,
      width: 200, height: 14,
      fillColor: '#e74c3c',
      trackColor: '#333333',
      value: 1.0,
    })
    this.ui.add(this._hp)

    this._scoreLabel = new LabelWidget({
      anchor: 'top-right',
      x: 16, y: 16,
      text: 'Score: 0',
      fontSize: 20,
      color: '#ffffff',
    })
    this.ui.add(this._scoreLabel)
  }

  override onUpdate(dt: number): void {
    this.ui.update(dt)
  }

  override onDestroy(): void {
    this.ui.clear()
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
import { Scene, ButtonWidget } from '@emptysock/engine'

class MenuScene extends Scene {
  override async onLoad(): Promise<void> {
    const btn = new ButtonWidget({
      label: 'Retry',
      anchor: 'center',
      width: 120,
      height: 40,
    })
    btn.on('click', () => this.engine.loadScene('GameScene'))
    btn.animate('fadeIn', { duration: 0.3 })
    this.ui.add(btn)
  }
}
```

---

## Compound panels

Build complex layouts by pushing children onto a `PanelWidget`. Children position relative to the panel's origin:

```typescript
import { Scene, PanelWidget, LabelWidget, ButtonWidget } from '@emptysock/engine'

class GameOverScene extends Scene {
  override async onLoad(): Promise<void> {
    const panel = new PanelWidget({
      anchor: 'center',
      width: 300, height: 200,
      background: '#1a1a2e',
    })

    const title = new LabelWidget({ text: 'Game Over', fontSize: 24, anchor: 'top', y: 16 })
    const score = new LabelWidget({ text: 'Score: 0', fontSize: 18, anchor: 'center' })
    const retry = new ButtonWidget({ label: 'Retry', anchor: 'bottom', y: 20, width: 100, height: 36 })

    retry.on('click', () => this.engine.loadScene('GameScene'))
    panel.children.push(title, score, retry)
    this.ui.add(panel)
    panel.animate('fadeIn')
  }
}
```

---

## UISystem API

Access via `scene.ui` (or `this.ui` inside a `Scene` subclass).

| Method | Description |
|--------|-------------|
| `ui.add(widget)` | Add a root widget to the overlay |
| `ui.remove(widget)` | Remove a specific root widget |
| `ui.clear()` | Remove all widgets |
| `ui.update(dt, px?, py?, cw?, ch?)` | Tick animations and hover state each frame |
| `ui.render(ctx, cw, ch)` | Draw all widgets to a Canvas 2D context |
| `ui.setImageLoader(loader)` | Override the default fetch-based image loader |
| `ui.handleClick(x, y, cw, ch)` | Hit-test and dispatch click; returns true if a widget was hit |
| `ui.handlePointerMove(x, y, cw, ch)` | Update hover state |

---

## Frame loop integration

```typescript
class MyScene extends Scene {
  override async onLoad(): Promise<void> {
    const btn = new ButtonWidget({ label: 'Play', anchor: 'center' })
    btn.animate('fadeIn', { duration: 0.3 })
    this.ui.add(btn)
  }

  override onUpdate(dt: number): void {
    this.ui.update(dt)
    // ui.render() is called automatically after PixiJS renders.
    // Only call it manually if you manage a raw Canvas 2D context yourself.
  }

  override onDestroy(): void {
    this.ui.clear()
  }
}
```

---

## Rules

- Each `Scene` has its own `UISystem` at `this.ui`. Never import or reference the class directly to get a shared instance — each scene is isolated.
- Never call `this.ui.add()` inside `onUpdate()` — create widgets in `onLoad()`, update properties in `onUpdate()`.
- Call `this.ui.clear()` in `onDestroy()` — or widgets linger until the next scene's clear call.
- Use `widget.visible = false` to hide temporarily; `this.ui.remove(w)` to fully remove.
- `widget.children` is a plain mutable array — push onto a `PanelWidget` to nest widgets.
- Never use definite-assignment `!` on widget fields; use `T | null = null` and check before use.
