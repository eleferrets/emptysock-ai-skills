# Story Graph / VNSystem

The Story Graph is the EmptySock panel for authoring branching dialogue trees (visual novels, cutscenes, quest dialogue). Scripts are exported as `.vnscript` JSON and played at runtime by `VNSystem`.

---

## Opening the Story Graph panel

In the IDE menu bar: **Module → Story Graph**. The panel is an SVG node graph editor.

---

## Node types

| Type | Purpose | Ports |
|------|---------|-------|
| `dialogue` | Speaker + text body | 1 input, 1 output |
| `choice` | Array of options | 1 input, N outputs (one per option) |
| `condition` | Branch on a variable value | 1 input, `true` output, `false` output |

---

## Authoring workflow

1. Right-click the canvas → **Add Node → Dialogue / Choice / Condition**.
2. Double-click a node to open its edit modal.
3. Drag from an output port to an input port to connect nodes.
4. Click **Export JSON** in the toolbar to save the graph as `story.vnscript`.
5. Place the file under `apps/ide/public/assets/story/`.
6. Load with `VNSystem.loadScript()` at runtime.

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
  type VNNode,
  type VNDialogueNode,
  type VNChoiceNode,
} from '@emptysock/engine'

// Create and load in onLoad:
const vn = new VNSystem()
await vn.loadScript('assets/story/chapter1.vnscript')

// Register node callback — called whenever the active node changes:
vn.onNode((node: VNNode) => {
  if (node.type === 'dialogue') {
    const d = node as VNDialogueNode
    showText(d.speaker, d.text)
  } else if (node.type === 'choice') {
    const c = node as VNChoiceNode
    showChoiceButtons(c.options.map((o) => o.label))
  }
})

// Begin playback:
vn.play()

// Advance a dialogue node:
vn.advance()

// Select a choice (zero-indexed):
vn.choose(0)

// Skip auto-advance delay:
vn.skip()

// Jump to a node by id (save/resume):
vn.jumpToNode('some-node-id')

// Variables (for Condition nodes):
vn.setVariable('bossDefeated', true)
const val = vn.getVariable('bossDefeated')       // boolean | string | number | undefined
const all  = vn.getVariables()                    // Record<string, string | number | boolean>

// Destroy with scene:
vn.destroy()
```

---

## Save and resume pattern

```typescript
import { SaveSystem, VNSystem } from '@emptysock/engine'
import { z } from 'zod'

const Schema = z.object({
  nodeId:    z.string(),
  variables: z.record(z.union([z.string(), z.number(), z.boolean()])),
})

// Save at each dialogue node:
function saveProgress(vn: VNSystem, nodeId: string): void {
  SaveSystem.save('vn-progress', {
    nodeId,
    variables: vn.getVariables(),
  }).catch(() => undefined)
}

// Resume on load:
async function tryResume(vn: VNSystem): Promise<void> {
  const slots = await SaveSystem.listSlots()
  if (!slots.includes('vn-progress')) return
  try {
    const raw  = await SaveSystem.load('vn-progress')
    const data = Schema.parse(raw.data)
    for (const [k, v] of Object.entries(data.variables)) {
      vn.setVariable(k, v)
    }
    vn.jumpToNode(data.nodeId)
  } catch {
    // corrupt save — start from beginning
  }
}
```

---

## Localisation pattern

Story text keys can be i18n keys resolved at display time:

```typescript
import { i18n } from '@emptysock/engine'

// In onNode callback for dialogue:
const text = i18n.t(node.text)   // returns key verbatim if missing — never throws
showText(node.speaker, text)
```

Use the LocalisationEditor panel to manage keys. Export JSON locale files to `src/locales/` and load them in `onLoad` with `i18n.load()`.

---

## Swapping music on story branches

Use a Condition node to set a variable, then read it in the node callback:

```typescript
vn.onNode((node) => {
  const chapter = vn.getVariable('chapter')
  if (chapter === 'inn') {
    Audio.music('inn_music', { loop: true, fade: 1.0 })
  } else if (chapter === 'forest') {
    Audio.music('forest_music', { loop: true, fade: 1.0 })
  }
})
```

---

## Rules

- `VNSystem` must be destroyed in `onDestroy` with `vn.destroy()`.
- Do not call `vn.advance()` or `vn.choose()` before `vn.play()` — the internal state machine is not ready.
- `onNode` fires once per node, including after `jumpToNode()`. Register the callback before calling `play()`.
- Condition nodes are evaluated and routed automatically — `onNode` is never called for them.
- Never cast `raw.data as MySaveType` — always use Zod to validate save data.
