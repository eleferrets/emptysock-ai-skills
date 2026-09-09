# Skill 20 — UI Widgets

Use this skill when building screen-space UI: menus, HUD elements, dialogue boxes, settings panels.

---

## When to use UISystem vs PixiJS

- **UISystem** — screen-space UI that stays in front of the game world at a fixed position (health bars on the HUD, pause menus, buttons).
- **PixiJS scene** — in-world UI that lives in game space (floating health bars above enemies, dialogue bubbles attached to NPCs, damage numbers).

---

## Widget classes

```ts
import {
  LabelWidget, ImageWidget, ButtonWidget, PanelWidget,
  ProgressBarWidget, SliderWidget, CheckboxWidget,
} from '@emptysock/engine';
```

All widgets share base properties and methods:

```ts
widget.x            // pixel offset from anchor
widget.y
widget.width
widget.height
widget.anchor       // 'top-left' | 'top' | 'top-right' | 'left' | 'center' | 'right' | 'bottom-left' | 'bottom' | 'bottom-right'
widget.visible
widget.alpha
widget.children     // Widget[] — mutable; add children with panel.children.push(child)
widget.on('click', handler)
widget.off('click', handler)
widget.animate('fadeIn', { duration: 300 })
```

---

## UIScene pattern — reusable overlay

Define a pause menu (or any overlay) as a Scene subclass. Push it on top; pop to return.

```ts
import { Scene, PanelWidget, LabelWidget, ButtonWidget } from '@emptysock/engine';

class PauseMenuScene extends Scene {
  private _panel: PanelWidget | null = null;

  onLoad(): void {
    const panel = new PanelWidget({ anchor: 'center', width: 300, height: 200 });
    this._panel = panel;

    const title = new LabelWidget({ text: 'Paused', fontSize: 24, anchor: 'top', y: 16 });

    const resumeBtn = new ButtonWidget({ label: 'Resume', anchor: 'center', y: 20 });
    resumeBtn.on('click', () => this.engine.popScene());

    const quitBtn = new ButtonWidget({ label: 'Quit to Menu', anchor: 'center', y: 70 });
    quitBtn.on('click', () => this.engine.loadScene('MainMenu'));

    panel.children.push(title, resumeBtn, quitBtn);
    this.uiSystem.add(panel);
    panel.animate('fadeIn');
  }

  onDestroy(): void {
    if (this._panel !== null) {
      this.uiSystem.removeWidget(this._panel);
    }
  }
}

// In a game scene, on Escape keypress:
this.engine.pushScene(new PauseMenuScene('pause'));
```

---

## Reusable widget factory pattern

For widgets shared across many scenes, a factory function is enough — no new class needed.

```ts
import { ButtonWidget } from '@emptysock/engine';

export function PrimaryButton(label: string, onClick: () => void): ButtonWidget {
  const btn = new ButtonWidget({ label, background: '#818cf8', width: 160, height: 40 });
  btn.on('click', onClick);
  return btn;
}
```

---

## Widget event types

| Event | Fired when |
|-------|-----------|
| `'click'` | Widget is clicked / tapped |
| `'hover'` | Pointer enters widget bounds |
| `'hoverOut'` | Pointer leaves widget bounds |
| `'change'` | Value changes (Slider, Checkbox) |
| `'animEnd'` | Animation finishes |

---

## Animation reference

```ts
widget.animate('fadeIn',   { duration: 300 });
widget.animate('fadeOut',  { duration: 200 });
widget.animate('slideIn',  { duration: 300, direction: 'left' });  // left | right | up | down
widget.animate('slideOut', { duration: 200, direction: 'down' });
widget.animate('pop',      { duration: 150 });   // scale punch — automatic on ButtonWidget hover
widget.animate('shake',    { duration: 300 });   // horizontal jitter — good for error feedback
```

All animations accept `{ duration?: number, easing?: 'linear'|'ease-in'|'ease-out'|'ease-in-out', direction? }`.

---

## HUD elements — no scene needed

For persistent HUD overlays, add widgets directly:

```ts
import { ProgressBarWidget, LabelWidget } from '@emptysock/engine';

class GameScene extends Scene {
  private _hpBar: ProgressBarWidget | null = null;
  private _scoreLabel: LabelWidget | null = null;

  onLoad(): void {
    const hp = new ProgressBarWidget({
      anchor: 'bottom', x: 0, y: 20, width: 300, height: 12,
      value: 100, min: 0, max: 100, fillColor: '#ef4444',
    });
    this._hpBar = hp;
    this.uiSystem.add(hp);

    const score = new LabelWidget({
      anchor: 'top-right', x: 16, y: 16, text: '0', fontSize: 18,
    });
    this._scoreLabel = score;
    this.uiSystem.add(score);
  }

  onDestroy(): void {
    if (this._hpBar !== null) this.uiSystem.removeWidget(this._hpBar);
    if (this._scoreLabel !== null) this.uiSystem.removeWidget(this._scoreLabel);
  }
}
```

---

## CheckboxWidget and SliderWidget

```ts
import { CheckboxWidget, SliderWidget } from '@emptysock/engine';

const mute = new CheckboxWidget({
  label: 'Mute audio', checked: false,
  onChange: (v) => { Audio.setGroupVolume('master', v ? 0 : 1); },
});

const vol = new SliderWidget({
  anchor: 'center', y: 40, width: 200,
  value: 0.8, min: 0, max: 1,
  onChange: (v) => { Audio.setGroupVolume('master', v); },
});
```

---

## Common mistakes

| Wrong | Right |
|-------|-------|
| Forgetting `onDestroy` cleanup | Always `uiSystem.removeWidget(widget)` in `onDestroy` |
| Mutating `widget.children` is not reactive | Re-add widget to UISystem after structural changes |
| Using UISystem for in-world UI | Use PixiJS scene objects for in-world positions |
| `panel.children = [btn]` | `panel.children.push(btn)` — array is readonly ref |
