# VNTextbox

**Use this when** you need a ready-made dialogue box for a Story Graph scene instead of building your own from `UISystem` widgets. `VNTextbox` is a pre-built panel anchored to the bottom of the canvas, with a speaker name plate and a text area. Call `bind(vn)` to wire it to a `VNSystem` instance — from there it updates itself whenever the current node changes, and clicking it calls `vn.advance()` for you.

## Import

`VNTextbox` and `VNSystem` both live in the optional `@emptysock/vn` package (see `skills/08-story-graph.md`), not the core engine:

```typescript
import { VNTextbox, VNSystem, type VNTextboxOptions } from '@emptysock/vn'
```

## Quick start

```typescript
class NarrativeScene extends Scene {
  private _vn: VNSystem | null = null
  private _textbox: VNTextbox | null = null

  override async onLoad(): Promise<void> {
    const response = await fetch('assets/story/chapter1.storyGraph.json')
    const graph = await response.json()
    const tree = storyGraphToDialogueTree(graph)

    this._vn = new VNSystem()

    this._textbox = new VNTextbox({
      ui: this.ui,          // pass the scene's UISystem instance
      canvasWidth:  800,
      canvasHeight: 600,
    })
    this._textbox.bind(this._vn)   // sync immediately to current node

    this._vn.load(tree)   // fires onNode for the first node immediately
  }

  override onUpdate(dt: number): void {
    this.ui.update(dt)
  }

  override onDestroy(): void {
    this._textbox?.destroy()   // removes widgets from this.ui
  }
}
```

## Constructor options

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `ui` | `UISystem` | required | The scene's UISystem instance — pass `this.ui` |
| `canvasWidth` | `number` | required | Canvas pixel width |
| `canvasHeight` | `number` | required | Canvas pixel height |
| `height` | `number` | `160` | Dialogue panel height in px |
| `namePlateHeight` | `number` | `36` | Speaker name plate height in px |
| `paddingX` | `number` | `24` | Horizontal padding inside the panel |
| `panelColor` | `string` | `"rgba(13,13,26,0.88)"` | CSS colour for panel fill |
| `namePlateColor` | `string` | `"#3c2d6e"` | CSS colour for name plate fill |
| `textColor` | `string` | `"#ffffff"` | Dialogue text colour |
| `fontSize` | `number` | `16` | Dialogue text size in px |

## API

| Method / Property | Signature | Notes |
|---|---|---|
| `bind` | `(vn: VNSystem): void` | Wire to a VNSystem; immediately syncs to the current node. |
| `visible` | `boolean` (getter/setter) | Show or hide the textbox. Auto-managed by node type. |
| `destroy` | `(): void` | Remove widgets from the UISystem. Call in onDestroy. |

## Advancing and choosing

Clicking anywhere on the textbox calls `vn.advance()` automatically for dialogue nodes. For choice nodes, the textbox renders options as numbered text (`1. Option A\n2. Option B`) — selection requires your own buttons:

```typescript
// In onLoad, after binding:
this._vn.setListener({
  onChoice: (options) => {
    options.forEach((opt, i) => {
      const btn = createChoiceButton(i + 1, opt.label)
      btn.on('click', () => {
        this._vn?.selectOption(opt.next)   // opt.next is the target node ID
        removeChoiceButtons()
      })
      this.ui.add(btn)
    })
  },
})
```

## Visibility

VNTextbox sets `visible` automatically based on the current node:
- `dialogue` or `choice` nodes → visible
- `event`, `jump`, `variable-set`, or null → hidden

## .storyGraph ↔ DialogueTree round-trip

```typescript
import { storyGraphToDialogueTree, dialogueTreeToStoryGraph, type StoryGraph, type DialogueTree } from '@emptysock/vn'

// Story Graph JSON (editor format) → runtime format:
const tree: DialogueTree = storyGraphToDialogueTree(graph)
vn.load(tree)

// Runtime format → Story Graph (re-import into the IDE editor):
const graph: StoryGraph = dialogueTreeToStoryGraph(tree)
```

## Rules

- Always pass `ui: this.ui` — VNTextbox needs a UISystem instance to hang its widgets on.
- Call `destroy()` in `onDestroy()`. Widgets don't remove themselves.
- Bind before calling `vn.load()` if you want the textbox showing the first node right away.
- VNSystem has no `destroy()` — just drop the reference and let it get garbage-collected.
