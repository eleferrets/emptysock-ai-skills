# Visual Script Editor

**Use this when** you're authoring node-graph logic in the IDE panel itself, rather than driving the graph runtime from code (see `skills/29-visual-script-component.md` for that side). The panel has two tabs:

- **Logic Script**: a node graph over `VariableStore` variables/switches and actor messages. This is the graph shape the runtime compiles and runs (`VisualScriptGraph`).
- **Scene Scaffold**: a scene/entity/component structure editor that can generate scaffold code (**Export Code**). It is separate from the runtime graph format.

---

## Logic Script canvas

| Action | Input |
|--------|-------|
| Add a node | The **Add node** toolbar control |
| Connect ports | Click an output port, then an input port |
| Delete selected node | Select it, then `Delete` or `Backspace` (or the delete button in the node's inspector) |
| Undo / redo | `Ctrl+Z` / `Ctrl+Shift+Z` (`Cmd` on macOS) |
| Zoom | `Ctrl`/`Cmd` + scroll wheel |
| Preview | The **Preview** / **Stop** toolbar button runs the graph against a throwaway `VariableStore`; fire an `onEvent` node with the event button |

---

## Logic Script nodes

These are exactly the runtime node kinds (`VSNodeKind`):

| Node | Ports | Fields |
|------|-------|--------|
| On Update | output | none (entry point, every update) |
| On Event | output | event type (entry point) |
| Sequence | input, output | none |
| Branch | input, `true` / `false` outputs | variable index, comparator (`eq`, `neq`, `gt`, `lt`, `gte`, `lte`), value |
| Get Variable | input, output | variable index, output key |
| Set Variable | input, output | variable index, literal value or a scoped key from an earlier Get Variable |
| Get Switch | input, output | switch index, output key |
| Set Switch | input, output | switch index, true/false |
| Send Message | input, output | target actor id, message type, payload |

There are no entity, component, math, loop, or audio nodes in this graph format. See `skills/36-visual-script-compiler.md` for why.

---

## Runtime

The graph the panel authors is a `VisualScriptGraph` (`{ nodes, connections }`). At runtime register it with `registerVisualScriptGraph(id, graph)`, attach `VisualScriptState` (`graphId`) to an entity, and drive it with `VisualScriptSystem`; see `skills/29-visual-script-component.md`. The same shape can be hand-authored in code with `VisualScriptGraphBuilder`.

---

## Performance guidance

Graphs are compiled once per `graphId` into JavaScript (`skills/36-visual-script-compiler.md`), so dispatch overhead is small, but a graph is still a poor fit for heavy per-frame computation. It suits event-driven, low-frequency logic: cutscene triggers, dialogue gating, puzzle switches.

---

## Tips

- Pair an `On Event` node with `VisualScriptSystem.fireEvent(scene, eventType)` from TypeScript, and a `Send Message` node with an `Actor` registered under the target id, to bridge graphs and the Actor Model.
- Graphs are plain JSON, so they diff and review in version control like any other file.
