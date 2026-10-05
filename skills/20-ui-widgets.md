# Skill 20 — UI Widget Patterns

**Use this when** you're building screen-space UI beyond a single widget (menus, HUD layouts, dialogue boxes, settings panels) and want patterns for wiring several widget entities together. API reference for the components lives in `skills/11-ui-system.md`.

---

## UISystem vs in-world objects

- **UISystem / WidgetTree**: screen-space UI at a fixed position (HUD bars, pause menus, buttons).
- **Entities with `Transform` + `Sprite`**: in-world UI in game space (health bars above enemies, damage numbers).

---

## Panel with children (pause menu)

```ts
import {
  defineScene, WidgetTree, UISystem, LayoutStyle, PanelStyle, Label, ButtonState,
} from '@emptysock/engine'

const tree = new WidgetTree()
const ui = new UISystem(tree)

function setStyle(e: ReturnType<typeof tree.createWidget>, patch: Partial<{ width: number; height: number; padding: number; gap: number }>): void {
  const s = e.get(LayoutStyle)
  if (s !== undefined) Object.assign(s, patch)
}

export const PauseMenu = defineScene({
  async onLoad(scene) {
    await tree.init()
    const panel = tree.createWidget(scene)
    setStyle(panel, { width: 300, height: 200, padding: 16, gap: 12 })
    panel.add(PanelStyle, { background: '#1a1a2e', borderRadius: 8 })

    const title = tree.createWidget(scene, panel)
    setStyle(title, { height: 32 })
    title.add(Label, { text: 'Paused', fontSize: 24, align: 1 })

    const resume = tree.createWidget(scene, panel)
    setStyle(resume, { height: 44 })
    resume.add(ButtonState, { label: 'Resume' })
  },
  onUnload(scene) {
    tree.destroy()
  },
})
```

Load this as an overlay (`await game.loadOverlay(PauseMenu)`) and unload it with `await game.unloadOverlay()`.

---

## Reusable widget factory

A plain function is enough; no class needed.

```ts
import { ButtonState, LayoutStyle, type Entity, type Scene, type WidgetTree } from '@emptysock/engine'

export function primaryButton(tree: WidgetTree, scene: Scene, parent: Entity, label: string): Entity {
  const e = tree.createWidget(scene, parent)
  const s = e.get(LayoutStyle)
  if (s !== undefined) { s.width = 160; s.height = 44 }
  e.add(ButtonState, { label, background: '#818cf8' })
  return e
}
```

---

## HUD elements

```ts
import { Progress, Label } from '@emptysock/engine'

const hpBar = tree.createWidget(scene, root)
hpBar.add(Progress, { value: 100, min: 0, max: 100, fillColor: '#ef4444' })

const scoreLabel = tree.createWidget(scene, root)
scoreLabel.add(Label, { text: '0', fontSize: 18, align: 2 })

// Later, in onUpdate:
const bar = hpBar.get(Progress)
if (bar !== undefined) bar.value = player.hp
```

---

## Checkbox and Slider

```ts
import { Checkbox, Slider } from '@emptysock/engine'

const mute = tree.createWidget(scene, root)
mute.add(Checkbox, { label: 'Mute audio', checked: false })

const vol = tree.createWidget(scene, root)
vol.add(Slider, { value: 0.8, min: 0, max: 1, step: 0.05 })

// Read state each frame (there are no onChange callbacks):
const box = mute.get(Checkbox)
const slider = vol.get(Slider)
if (box !== undefined && slider !== undefined) game.audio.setGroupVolume('master', box.checked ? 0 : slider.value)
```

---

## Common mistakes

| Wrong | Right |
|-------|-------|
| Forgetting to `await tree.init()` | `layout()` throws until Yoga has loaded |
| Expecting `.on('click')` or animation helpers | `UISystem` has no events or animations; poll component state and tween fields yourself (see `skills/19-tweens.md`) |
| Destroying a parent and expecting children to go too | `WidgetTree.destroyWidget` does not cascade; destroy children first |
| Using UISystem for in-world UI | Use `Transform` + `Sprite` entities for anything positioned in game space |
