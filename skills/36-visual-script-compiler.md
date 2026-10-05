# Visual scripting compiler

**Use this when** you're working out what actually runs a visual-script graph. For the runtime API see `skills/29-visual-script-component.md`; for the editor panel (canvas controls, node palette) see `skills/visual-script.md`.

---

## Graphs are always compiled

`VisualScriptSystem` compiles each distinct `graphId` once, via `compileVisualScriptGraph(graph)`, into JavaScript source and caches the resulting module for the system's lifetime. Many entities sharing one `graphId` share one compiled module. There is no separate interpreter class and no opt-in compiled variant.

```typescript
import { compileVisualScriptGraph } from '@emptysock/engine'

const source: string = compileVisualScriptGraph(graph)   // JS source exporting run(ctx) and fireEvent(eventType, ctx)
```

You rarely need that directly; `VisualScriptSystem.update(scene)` / `fireEvent(scene, type)` do it for you.

## What "compiles" means

The compiler emits a `switch` inside a `while` loop keyed on node id, because a graph may legally contain a cycle and straight-line code cannot represent one. Each `case` is code specific to its node with field values baked in as literals. A 10,000-step-per-trigger cap guards against runaway cycles.

## Node vocabulary is fixed

Compiled graphs only use the editor's Logic Script vocabulary: `VariableStore` get/set variable/switch and `ActorSystem.send`. There is no spawn-entity or add-component node. Adding one is new node-kind work in both the graph types and the compiler, not something to fake with an existing node.

## Cache invalidation

Re-registering a graph with `registerVisualScriptGraph(id, newGraph)` does not rebuild a cached module; call `visualScriptSystem.invalidate(id)` afterwards.
