# VNTextbox and VNBackgroundLayer

**Use this when** you need a ready-made dialogue box for a Story Graph scene, or background/CG image layers for a visual novel. Both live in the optional `@emptysock/vn` package (see `skills/08-story-graph.md`). `VNTextbox` is built from widget entities (`PanelStyle`, `Label`) spawned through a `WidgetTree`; see `skills/11-ui-system.md` for the widget model.

## Import

```typescript
import { VNTextbox, VNSystem, VNBackgroundLayer, type VNTextboxOptions } from '@emptysock/vn'
```

## Quick start

```typescript
import { defineScene, WidgetTree, UISystem } from '@emptysock/engine'
import { VNSystem, VNTextbox, storyGraphToDialogueTree, type StoryGraph } from '@emptysock/vn'

const tree = new WidgetTree()
const ui = new UISystem(tree)
let textbox: VNTextbox | null = null
let vn: VNSystem | null = null

export const Narrative = defineScene({
  async onLoad(scene, ctx) {
    await tree.init()
    const graph = (await (await fetch('assets/story/chapter1.storyGraph.json')).json()) as StoryGraph

    vn = new VNSystem(ctx.variables)
    textbox = new VNTextbox({ canvasWidth: 800, canvasHeight: 600, scene, tree, typewriterSpeed: 40 })
    textbox.bind(vn)                                  // sync to the current node
    vn.load(storyGraphToDialogueTree(graph))          // fires onNode for the first node
    textbox.bind(vn)                                  // re-sync after load so the first line shows
  },
  onUpdate(dt) {
    textbox?.update(dt)                               // advances the typewriter
    // each frame: tree.layout(scene, 800, 600); then ui.render(scene, ctx2d)
  },
  onUnload() {
    textbox?.destroy()
    tree.destroy()
  },
})
```

Pointer input is yours to route: call `textbox.handlePointerDown(x, y)` from your pointer handler. It returns whether the panel was hit, skips an in-progress typewriter on the first click, and advances the VN on the next.

## Constructor options (`VNTextboxOptions`)

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `canvasWidth` | `number` | required | Canvas pixel width |
| `canvasHeight` | `number` | required | Canvas pixel height |
| `scene` | `Scene` | required | Scene the widget entities are spawned into |
| `tree` | `WidgetTree` | required | Widget tree that owns the entities |
| `height` | `number` | `160` | Dialogue panel height in px |
| `namePlateHeight` | `number` | `36` | Speaker name plate height in px |
| `paddingX` | `number` | `24` | Horizontal padding inside the panel |
| `panelColor` | `string` | `"#0d0d1a"` | CSS colour for panel fill |
| `namePlateColor` | `string` | `"#3c2d6e"` | CSS colour for name plate fill |
| `textColor` | `string` | `"#ffffff"` | Text colour |
| `fontSize` | `number` | `16` | Text size in px |
| `fontFamily` | `string` | `"sans-serif"` | Family used to pre-wrap lines for the typewriter |
| `typewriterSpeed` | `number` | off | Characters per second; 0 or omitted = instant |

## API

| Member | Signature | Notes |
|---|---|---|
| `bind` | `(vn: VNSystem): void` | Wire to a VNSystem; syncs to the current node |
| `update` | `(dt: number): void` | Advance the typewriter; call every frame |
| `skipTypewriter` | `(): void` | Jump to the end of the current reveal |
| `isTyping` | `boolean` getter | True while revealing |
| `handlePointerDown` | `(x: number, y: number): boolean` | Hit-test the laid-out panel; advances/skips |
| `visible` | `boolean` getter/setter | Show or hide the panel |
| `destroy` | `(): void` | Destroys its widget entities; call on scene unload |

`bind` only syncs when called; after `vn.load()` or when `VNSystem` moves to a new node, `handlePointerDown` re-syncs for you on advance. Call `bind(vn)` again after `load()` as in the example if you want the first line shown immediately.

## Choices

For choice nodes the textbox renders options as numbered text (`1. Option A`). Selection needs your own UI or input handling:

```typescript
vn.setListener({
  onChoice: (options) => {
    // show your own buttons (see skills/11-ui-system.md), then on pick:
    //   vn.selectOption(options[i].next)
  },
})
```

## Visibility

- `dialogue` or `choice` node: panel visible.
- Any other node type, or no current node: panel hidden.

---

## VNBackgroundLayer

A Canvas 2D background plus CG layer with cross-fades.

```typescript
const bg = new VNBackgroundLayer({ canvasWidth: 800, canvasHeight: 600, fadeDuration: 0.5 })
bg.setBackground('assets/bg/forest.png', { fit: 'cover' })   // fit: 'cover' | 'contain' | 'stretch'
bg.showCG('assets/cg/forest_encounter.jpg')                  // CG default fit: 'contain'

// each frame:
bg.update(dt)
bg.render(ctx2d)      // CanvasRenderingContext2D; draw before the UI
```

| Method | Signature |
|--------|-----------|
| `setBackground` | `(imagePath, opts?: { fadeDuration?, fit? }): void` |
| `clearBackground` | `(fadeDuration?): void` |
| `showCG` | `(imagePath, opts?: { fadeDuration?, fit? }): void` |
| `hideCG` | `(fadeDuration?): void` |
| `update` | `(dt: number): void` |
| `render` | `(ctx: CanvasRenderingContext2D): void` |

## Rules

- Call `destroy()` on the textbox when the scene unloads; widget entities don't remove themselves.
- Await `tree.init()` and call `tree.layout(...)` each frame (or after changes) before rendering.
- `VNSystem` has no `destroy()`; drop the reference.
