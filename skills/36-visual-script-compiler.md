# Visual scripting compiler

**Use this when** you're working with `VisualScriptComponent`, the Visual Script Editor's Logic Script graphs, or `CompiledVisualScriptComponent`. For the editor panel itself (canvas controls, node palette, `.esvs` format), see `skills/visual-script.md`; this file is about what actually runs the graph.

---

## Two ways to run the same graph, same behaviour, your choice

`VisualScriptComponent` interprets a graph node-by-node at runtime — this is the default, and it's what you get if you don't think about this decision at all. `CompiledVisualScriptComponent` is an opt-in, same-shape drop-in that instead compiles the graph once into real JavaScript and runs that. Both are tested against each other node-by-node, so they can never quietly drift apart in behaviour — pick compiled if you've profiled and the interpreter's dispatch overhead actually matters for your game, otherwise don't bother.

## What "compiles" actually means here

The compiler turns a graph into a `switch` statement inside a `while` loop, keyed on node id — not straight-line code, because a graph is allowed to contain a cycle (an actual, legal use case: a patrol loop, a repeating dialogue check), and straight-line code can't represent a loop. Each `case` in that switch is real, specific code for that one node with its field values baked in as literals at compile time — `variables.setVar(1, 5)`, not a generic "read `node.kind` and dispatch at runtime" call. Only the *loop shape* is shared with the interpreter; the actual per-node logic is genuinely compiled, not reinterpreted through another layer.

## It targets the graph you already have, not a new one

The compiler emits calls against the same node vocabulary the Visual Script Editor's Logic Script tab already authors today — `VariableStore` get/set var/switch, `ActorSystem.send`. There's no entity/component or `scene.spawn` node kind in this graph format; if you're picturing "drag a node to spawn an enemy," that's not what this compiles — it's the message-passing/variable-logic graph format, full stop. Extending the node vocabulary to cover spawn/component operations would be a real, separate piece of work, not something you get by switching from interpreted to compiled.

## Why this matters for you as an agent

If someone asks you to "make the visual script faster," the answer is almost always "switch this one component from `VisualScriptComponent` to `CompiledVisualScriptComponent`," not a rewrite of their graph logic. If someone asks you to add a node type that doesn't exist yet (spawn an entity, add a component), that's new node-kind work on the interpreter and the compiler both — say so plainly rather than trying to fake it with an existing node.
