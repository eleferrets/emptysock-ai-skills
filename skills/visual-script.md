# Visual Script Editor

The Visual Script Editor is an IDE panel for wiring component logic without writing TypeScript. It produces `.esvs` files that run through a graph interpreter at runtime.

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

`Ctrl+S` or the **Save** toolbar button writes a `.esvs` JSON file. Reference it from a `VisualScriptComponent` on any entity:

```typescript
import { VisualScriptComponent } from '@emptysock/engine';

entity.addComponent(VisualScriptComponent, { script: 'assets/scripts/door-logic.esvs' });
```

---

## Performance guidance

Visual scripts run through a graph interpreter — expect roughly 10× slower execution than native TypeScript for hot paths. Use visual scripts for event-driven, low-frequency logic: cutscenes, dialogue triggers, UI flows, puzzle mechanics. Move anything that runs every frame with heavy computation to a TypeScript scene or actor.

---

## Tips

- Organise complex graphs into sub-graphs: right-click selected nodes → Collapse to Subgraph.
- Use `Comment` nodes (right-click → Add Comment) to annotate intent for collaborators.
- The `On Message` node works with the Actor Model — pair it with `actor.send()` from TypeScript to bridge the two systems.
- Visual scripts are plain JSON — they can be diffed and reviewed in version control.
