# Story Graph / VNSystem

**Use this when** you're building branching dialogue, visual novel scenes, or cutscenes — anything the Story Graph panel authors. `VNSystem` and its helpers live in the optional `@emptysock/vn` package, not the core engine — install and import it only if the game actually has dialogue.

The Story Graph is the EmptySock IDE panel for authoring branching dialogue trees (visual novels, cutscenes, quest dialogue). Scripts are exported as `.storyGraph.json` (the editor format) and converted to `DialogueTree` at runtime by `storyGraphToDialogueTree()`, then played by `VNSystem`.

`VNSystem`'s constructor defaults to the engine's global `variableStore` singleton (`constructor(store: VariableStore = variableStore)`) — a VN choice gated on switch 12 shares state with anything else in the game touching that same switch. Usually what you want; pass your own `VariableStore` instance if you need an isolated store (e.g. per save slot, or in a test).

---

## Opening the Story Graph panel

In the IDE menu bar: **Module → Story Graph**. The panel is an SVG node graph editor.

---

## Node types

| Type | Purpose | Ports |
|------|---------|-------|
| `dialogue` | Speaker + text body | 1 input, 1 output |
| `choice` | Array of options, each optionally gated by a `VariableCondition` (`when`) | 1 input, N outputs (one per option) |
| `condition` | Branches to `ifTrue` or `ifFalse` based on a `VariableCondition` | 1 input, 2 outputs |

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
  type IVNListener,
  type DialogueNode,
  type DialogueTree,
  type StoryGraph,
} from '@emptysock/vn'

// In onLoad — fetch the exported graph, convert, and load:
override async onLoad(): Promise<void> {
  const response = await fetch('assets/story/chapter1.storyGraph.json')
  const graph: StoryGraph = await response.json() as StoryGraph

  const tree: DialogueTree = storyGraphToDialogueTree(graph)

  // Uses the shared `variableStore` singleton by default — pass a different
  // `VariableStore` instance as the constructor argument for isolated state
  // (per-save-slot, tests). This is what "condition" nodes and a choice
  // option's `when` field are evaluated against.
  const vn = new VNSystem()

  // Register a listener BEFORE calling load():
  vn.setListener({
    onNode(node: DialogueNode) {
      if (node.type === 'dialogue') {
        showText(node.speaker, node.text)
      }
    },
    onChoice(options) {
      // options: Array<{ label: string; next: string }>
      showChoiceButtons(options)
    },
    onEnd() {
      hideDialogueBox()
    },
  } satisfies IVNListener)

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

## Reading variables

`'variable-set'` nodes store values in `vn.variables` and auto-advance. Read them at any time:

```typescript
const metHero: unknown = vn.getVariable('metHero')
```

---

## DialogueNode type

`DialogueNode` is a discriminated union — narrow by `node.type`:

```typescript
vn.setListener({
  onNode(node: DialogueNode) {
    if (node.type === 'dialogue') {
      // node.speaker: string
      // node.text: string
      // node.next?: string (next node id, or undefined if last)
    } else if (node.type === 'choice') {
      // node.text: string (prompt text shown above options)
      // node.options: Array<{ label: string; next: string; when?: VariableCondition }>
      // Options whose `when` evaluates false are already filtered out before
      // onChoice fires — you never see them and never re-check the condition.
    }
    // 'jump' nodes are resolved automatically — onNode never fires for them
    // 'variable-set' nodes auto-advance; read via vn.getVariable(key)
    // 'condition' nodes resolve and advance to ifTrue/ifFalse synchronously,
    // like jump — onNode never fires for them either
  },
  onEvent(eventName, data) {
    // eventName: string — fire game logic
    // data: unknown — additional payload
    // auto-advanced by the engine
  },
})
```

---

## Branching on persistent variables

`'condition'` dialogue nodes and a choice option's `when` field both branch on the shared `VariableStore` (see `skills/13-variable-store.md`), so a Story Graph can react to what happened elsewhere in the game — a boss fight, a switch flipped by a map event — without any special-case code in the scene.

```typescript
import { variableStore } from '@emptysock/engine'
import { VNSystem, type DialogueTree } from '@emptysock/vn'

variableStore.setSwitchName(10, 'bossDefeated')

const tree: DialogueTree = {
  startNode: 'throneRoomCheck',
  nodes: {
    throneRoomCheck: {
      type: 'condition',
      condition: { kind: 'switch', index: 10, equals: true },
      ifTrue: 'kingThanksYou',
      ifFalse: 'kingWarnsYou',
    },
    kingThanksYou: { type: 'dialogue', speaker: 'King', text: 'You saved the realm.' },
    kingWarnsYou: { type: 'dialogue', speaker: 'King', text: 'The dragon still lives — hurry.' },
  },
}

const vn = new VNSystem()  // uses the shared variableStore by default
vn.setListener({
  onNode(node) {
    if (node.type === 'dialogue') showText(node.speaker, node.text)
  },
})
vn.load(tree)  // shows "The dragon still lives — hurry." while switch 10 is false
```

A choice option works the same way — add `when: { kind: 'variable', index: 4, op: 'gte', value: 10 }` to any option and it is silently excluded from `onChoice`'s array until the condition is true.

---

## Save and resume pattern

VNSystem has no internal save state. Store enough to recreate position yourself:

```typescript
import { SaveSystem } from '@emptysock/engine'
import { VNSystem, storyGraphToDialogueTree } from '@emptysock/vn'
import { z } from 'zod'

const progressSaves = new SaveSystem<{ id: string; nodeId: string }>(
  'vn_progress_',
  z.object({ id: z.string(), nodeId: z.string() }),
)

// Save the current node id:
function saveProgress(currentNodeId: string): void {
  progressSaves.save('vn-progress', { nodeId: currentNodeId })
}

// Resume: load the tree again and navigate to the saved node
// by walking the graph until reaching it, or by using selectOption to jump.
// The simplest pattern: call vn.load(tree), then advance/selectOption until
// currentNode?.type is not 'jump' and the id matches.
```

---

## Story Graph ↔ DialogueTree round-trip

```typescript
import { storyGraphToDialogueTree, dialogueTreeToStoryGraph, type StoryGraph, type DialogueTree } from '@emptysock/vn'

// Editor format → runtime format:
const tree: DialogueTree = storyGraphToDialogueTree(graph)

// Runtime format → editor format (for re-import into the Story Graph panel):
const graph: StoryGraph = dialogueTreeToStoryGraph(tree)
```

---

## Rules

- Register a listener with `setListener()` **before** calling `load()` — `onNode` fires immediately for the first node.
- `jump` and `variable-set` nodes are resolved automatically — `onNode` fires for `variable-set` but the engine auto-advances it; read the value with `getVariable()`.
- VNSystem has no `destroy()` — it is garbage-collected when the scene releases it.
- Never cast loaded JSON directly as `StoryGraph` without validation — use Zod in production.
