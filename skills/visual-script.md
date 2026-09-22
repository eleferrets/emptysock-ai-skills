# Visual Script Editor

**Use this when** you're authoring node-graph logic in the IDE panel itself, rather than driving `VisualScriptComponent` from code (see `skills/29-visual-script-component.md` for that side). The Visual Script Editor lets you wire up component logic without writing TypeScript, and produces `.esvs` files that a graph interpreter runs at runtime.

---

## Opening the panel

View → Panels → Visual Script Editor, or drag the tab from the tab bar into a docked pane.

---

## Canvas controls

| Action | Input |
|--------|-------|
| Pan | Middle-click drag, or Space + drag |
| Zoom | Scroll wheel |
| Select node | Click |
| Multi-select | Shift-click or drag a selection box |
| Move nodes | Drag selected nodes |
| Delete selected | `Delete` or `Backspace` |
| Connect ports | Drag from an output port to an input port |
| Disconnect a port | Click a connected port and drag off |
| Open node picker | Right-click canvas or `Tab` |

---

## Adding nodes

Right-click the canvas (or press `Tab`) to open the node picker. Node categories:

- **Entity** — `Get Entity`, `Create Entity`, `Destroy Entity`
- **Component** — `Add Component`, `Get Component`, `Set Property`, `Get Property`
- **Events** — `On Update`, `On Collision Enter`, `On Message`
- **Flow** — `Branch` (if/else), `Sequence`, `For Each`
- **Math** — `Add`, `Subtract`, `Multiply`, `Compare`, `Lerp`
- **Output** — `Log`, `Play Audio`, `Load Scene`

---

## Edge types

- **Yellow edges** — control-flow signals (execution order).
- **White edges** — data values (numbers, strings, component references).

Ports are colour-coded by type. Connecting incompatible types shows a red error indicator.

---

## Saving

`Ctrl+S` or the **Save** toolbar button writes a `.esvs` JSON file.

> **Note:** `VisualScriptComponent` is the runtime that attaches a graph to an entity via TypeScript and interprets it every `update()`/`fireEvent()` call — see `skills/29-visual-script-component.md` for its full API and `VisualScriptGraphBuilder` for hand-authoring the same graph shape in code.

---

## Performance guidance

Visual scripts run through a graph interpreter, so expect roughly 10x slower execution than native TypeScript on hot paths. They're a great fit for event-driven, low-frequency logic: cutscenes, dialogue triggers, UI flows, puzzle mechanics. Anything running every frame with real computation belongs in a TypeScript scene or actor instead.

---

## Tips

- Break up complex graphs into sub-graphs: right-click selected nodes → Collapse to Subgraph. Your future self will thank you.
- Use `Comment` nodes (right-click → Add Comment) to leave notes for whoever opens this graph next, including you in six months.
- The `On Message` node plugs into the Actor Model — pair it with `actor.send()` from TypeScript to bridge the two systems.
- Visual scripts are plain JSON, so they diff and review in version control just like any other file.
