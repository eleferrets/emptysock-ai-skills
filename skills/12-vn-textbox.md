# VNTextbox

`VNTextbox` is a pre-built dialogue box component rendered by `UISystem`. It creates a panel anchored to the bottom of the canvas with a speaker name plate and a text area, and wires itself to a `VNSystem` instance so no boilerplate is needed.

---

## Quick start

```typescript
import { VNSystem, VNTextbox, UISystem } from '@emptysock/engine'

class NarrativeScene extends Scene {
  private _vn = new VNSystem()
  private _textbox!: VNTextbox

  async onLoad(): Promise<void> {
    await this._vn.loadScript('assets/story/chapter1.vnscript')

    this._textbox = new VNTextbox({
      canvasWidth:  800,
      canvasHeight: 600,
    })
    this._textbox.bind(this._vn)
    this._vn.play()
  }

  onUpdate(dt: number): void {
    UISystem.update(dt)
    UISystem.render(ctx, 800, 600)
  }

  onDestroy(): void {
    this._textbox.unbind()
    this._vn.destroy()
  }
}
```

---

## Constructor options

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `canvasWidth` | `number` | required | Canvas pixel width |
| `canvasHeight` | `number` | required | Canvas pixel height |
| `height` | `number` | `160` | Dialogue panel height in px |
| `namePlateHeight` | `number` | `36` | Speaker name plate height in px |
| `paddingX` | `number` | `24` | Horizontal padding inside the panel |
| `panelColor` | `number` | `0x0d0d1a` | 0xRRGGBB panel fill |
| `namePlateColor` | `number` | `0x3c2d6e` | 0xRRGGBB name plate fill |
| `textColor` | `number` | `0xffffff` | Dialogue text colour |
| `fontSize` | `number` | `16` | Dialogue text size in px |

---

## API

```typescript
textbox.bind(vn: VNSystem)   // Wire to a VNSystem; immediately syncs to current node
textbox.unbind()             // Detach from VNSystem; call in onDestroy()
textbox.visible              // get/set — show or hide the textbox
```

---

## Advancing dialogue

Clicking anywhere on the textbox advances the current dialogue node via `VNSystem.advance()` automatically. Choice nodes show numbered options as text — selection must be driven through `VNSystem.choose()`:

```typescript
// Choice node detected — options are displayed as "1. Option A\n2. Option B"
// Wire choice buttons separately:
choiceBtn1.onClick(() => {
  this._vn.choose(0)   // select first option (zero-indexed)
})
choiceBtn2.onClick(() => {
  this._vn.choose(1)   // select second option
})
```

---

## .vnscript ↔ Story Graph round-trip

The IDE's Story Graph editor can export and import `VNSystem` `DialogueTree` JSON using `storyGraphToDialogueTree()` and `dialogueTreeToStoryGraph()`:

```typescript
import { storyGraphToDialogueTree, dialogueTreeToStoryGraph } from '@emptysock/engine'

// Story Graph (IDE format) → DialogueTree (runtime format)
const tree = storyGraphToDialogueTree({ nodes, edges, startNodeId: 'n1' })
// load via a .vnscript file written from this tree

// DialogueTree → Story Graph (for re-importing into the editor)
const graph = dialogueTreeToStoryGraph(tree)
// graph.nodes and graph.edges can be imported into the Story Graph panel
```

The IDE toolbar provides **Export .vnscript** and **Import .vnscript** buttons that perform this conversion automatically.

---

## Rules

- Call `unbind()` in `onDestroy()` — VNTextbox registers root components in `UISystem` and they persist until removed.
- Do not use `VNTextbox` without binding a `VNSystem` — calling `bind()` after creation is required for the box to update.
- The textbox's `visible` property is managed automatically based on `VNSystem` current node type: event and null nodes hide it.
