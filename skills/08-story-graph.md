# Story Graph / VNSystem

The Story Graph is the EmptySock IDE panel for authoring branching dialogue trees (visual novels, cutscenes, quest dialogue). Scripts are exported as `.storyGraph.json` (the editor format) and converted to `DialogueTree` at runtime by `storyGraphToDialogueTree()`, then played by `VNSystem`.

---

## Opening the Story Graph panel

In the IDE menu bar: **Module → Story Graph**. The panel is an SVG node graph editor.

---

## Node types

| Type | Purpose | Ports |
|------|---------|-------|
| `dialogue` | Speaker + text body | 1 input, 1 output |
| `choice` | Array of options | 1 input, N outputs (one per option) |

---

## Authoring workflow

1. Right-click the canvas → **Add Node → Dialogue / Choice**.
2. Double-click a node to open its edit modal.
3. Drag from an output port to an input port to connect nodes.
4. Click **Export** in the toolbar to download the graph as JSON.
5. Place the file under `assets/story/`.
6. At runtime, fetch the file, convert, and pass to `VNSystem.load()`.

**Canvas controls:**

| Action | Input |
|--------|-------|
| Pan | Middle-click drag, Space + drag, or two-finger trackpad swipe |
| Zoom | Scroll wheel or trackpad pinch |
| Move node | Drag the node header |
| Connect | Drag output port → input port |
| Edit node | Double-click node body |
| Delete | Select then `Delete` |

---

## VNSystem API

```typescript
import {
  VNSystem,
  storyGraphToDialogueTree,
  type DialogueNode,
  type DialogueTree,
  type StoryGraph,
} from '@emptysock/engine'

// In onLoad — fetch the exported graph, convert, and load:
override async onLoad(): Promise<void> {
  const response = await fetch('assets/story/chapter1.storyGraph.json')
  const graph: StoryGraph = await response.json() as StoryGraph

  const tree: DialogueTree = storyGraphToDialogueTree(graph)

  const vn = new VNSystem()

  // Register callbacks BEFORE calling load():
  vn.onNode = (node: DialogueNode) => {
    if (node.type === 'dialogue') {
      showText(node.speaker, node.text)
    }
  }

  vn.onChoice = (options) => {
    // options: Array<{ label: string; next: string }>
    showChoiceButtons(options)
  }

  vn.onEnd = () => {
    hideDialogueBox()
  }

  vn.load(tree)   // synchronous — fires onNode for the first node immediately
}
```

---

## Advancing dialogue

```typescript
// Advance past a dialogue node to the next:
vn.advance()

// Select a choice — pass the target node ID from the option:
// options come from onChoice callback: Array<{ label: string; next: string }>
function onChoiceSelected(next: string): void {
  vn.selectOption(next)
}
```

---

## DialogueNode type

`DialogueNode` is a discriminated union — narrow by `node.type`:

```typescript
vn.onNode = (node: DialogueNode) => {
  if (node.type === 'dialogue') {
    // node.speaker: string
    // node.text: string
    // node.next?: string (next node id, or undefined if last)
  } else if (node.type === 'choice') {
    // node.text: string (prompt text shown above options)
    // node.options: Array<{ label: string; next: string }>
  } else if (node.type === 'event') {
    // node.eventName: string — fire game logic
    // node.data?: Record<string, unknown>
    // auto-advanced by the engine after firing onEvent
  } else if (node.type === 'variable-set') {
    // node.variableKey: string
    // node.variableValue: unknown
    // auto-advanced by the engine after firing onNode
  }
  // 'jump' nodes are resolved automatically — onNode never fires for them
}
```

---

## Save and resume pattern

VNSystem has no internal save state. Store enough to recreate position yourself:

```typescript
import { SaveSystem, VNSystem, storyGraphToDialogueTree } from '@emptysock/engine'
import { z } from 'zod'

const Schema = z.object({ nodeId: z.string() })

// Save the current node id:
function saveProgress(currentNodeId: string): void {
  SaveSystem.save('vn-progress', { nodeId: currentNodeId }).catch(() => undefined)
}

// Resume: load the tree again and navigate to the saved node
// by walking the graph until reaching it, or by using selectOption to jump.
// The simplest pattern: call vn.load(tree), then advance/selectOption until
// currentNode?.type is not 'jump' and the id matches.
```

---

## Story Graph ↔ DialogueTree round-trip

```typescript
import { storyGraphToDialogueTree, dialogueTreeToStoryGraph, type StoryGraph, type DialogueTree } from '@emptysock/engine'

// Editor format → runtime format:
const tree: DialogueTree = storyGraphToDialogueTree(graph)

// Runtime format → editor format (for re-import into the Story Graph panel):
const graph: StoryGraph = dialogueTreeToStoryGraph(tree)
```

---

## Rules

- Register all callbacks (`onNode`, `onChoice`, `onEvent`, `onEnd`) **before** calling `load()` — `onNode` fires immediately for the first node.
- `jump` and `variable-set` nodes are resolved automatically — `onNode` is called for `variable-set` but the engine auto-advances it.
- VNSystem has no `destroy()` — it is garbage-collected when the scene releases it.
- Never cast loaded JSON directly as `StoryGraph` without validation — use Zod in production.
