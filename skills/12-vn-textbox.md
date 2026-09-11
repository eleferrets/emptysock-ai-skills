# VNTextbox

`VNTextbox` is a pre-built dialogue box rendered by `UISystem`. It creates a panel anchored to the bottom of the canvas with a speaker name plate and a text area. Call `bind(vn)` to wire it to a `VNSystem` instance — it updates automatically whenever the current node changes. Clicking the textbox calls `vn.advance()` automatically.

## Import

```typescript
import { VNTextbox, VNSystem, UISystem, type VNTextboxOptions } from '@emptysock/engine'
```

## Quick start

```typescript
class NarrativeScene extends Scene {
  private _vn!: VNSystem
  private _textbox!: VNTextbox

  override async onLoad(): Promise<void> {
    const response = await fetch('assets/story/chapter1.storyGraph.json')
    const graph = await response.json()
    const tree = storyGraphToDialogueTree(graph)

    this._vn = new VNSystem()

    this._textbox = new VNTextbox({
      canvasWidth:  800,
      canvasHeight: 600,
    })
    this._textbox.bind(this._vn)   // sync immediately to current node

    this._vn.onChoice = (options) => {
      // choice selection is external — VNTextbox doesn't provide choice buttons
      // wire your own buttons and call this._vn.selectOption(option.next)
    }

    this._vn.load(tree)   // fires onNode for the first node immediately
  }

  override onUpdate(dt: number): void {
    UISystem.update(dt)
  }

  override onDestroy(): void {
    this._textbox.destroy()   // removes UISystem components
  }
}
```

## Constructor options

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `canvasWidth` | `number` | required | Canvas pixel width |
| `canvasHeight` | `number` | required | Canvas pixel height |
| `height` | `number` | `160` | Dialogue panel height in px |
| `namePlateHeight` | `number` | `36` | Speaker name plate height in px |
| `paddingX` | `number` | `24` | Horizontal padding inside the panel |
| `panelColor` | `number` | `0x0d0d1a` | 0xRRGGBB panel fill (80% opacity) |
| `namePlateColor` | `number` | `0x3c2d6e` | 0xRRGGBB name plate fill |
| `textColor` | `number` | `0xffffff` | Dialogue text colour |
| `fontSize` | `number` | `16` | Dialogue text size in px |

## API

| Method / Property | Signature | Notes |
|---|---|---|
| `bind` | `(vn: VNSystem): void` | Wire to a VNSystem; immediately syncs to the current node. |
| `visible` | `boolean` (getter/setter) | Show or hide the textbox. Auto-managed by node type. |
| `destroy` | `(): void` | Remove components from UISystem. Call in onDestroy. |

## Advancing and choosing

Clicking anywhere on the textbox calls `vn.advance()` automatically for dialogue nodes. For choice nodes, the textbox renders options as numbered text (`1. Option A\n2. Option B`) — selection requires your own buttons:

```typescript
// In onLoad, after binding:
this._vn.onChoice = (options) => {
  options.forEach((opt, i) => {
    const btn = createChoiceButton(i + 1, opt.label)
    btn.onClick(() => {
      this._vn.selectOption(opt.next)   // opt.next is the target node ID
      removeChoiceButtons()
    })
  })
}
```

## Visibility

VNTextbox sets `visible` automatically based on the current node:
- `dialogue` or `choice` nodes → visible
- `event`, `jump`, `variable-set`, or null → hidden

## .storyGraph ↔ DialogueTree round-trip

```typescript
import { storyGraphToDialogueTree, dialogueTreeToStoryGraph, type StoryGraph, type DialogueTree } from '@emptysock/engine'

// Story Graph JSON (editor format) → runtime format:
const tree: DialogueTree = storyGraphToDialogueTree(graph)
vn.load(tree)

// Runtime format → Story Graph (re-import into the IDE editor):
const graph: StoryGraph = dialogueTreeToStoryGraph(tree)
```

## Rules

- Call `destroy()` in `onDestroy()` — VNTextbox registers root components in `UISystem` and they persist until removed.
- Bind before calling `vn.load()` if you want the textbox to show the first node immediately.
- VNSystem has no `destroy()` — release the reference and it is garbage-collected.
