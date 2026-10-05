# Visual script runtime (VisualScriptState + VisualScriptSystem)

**Use this when** you're running a visual-script graph at runtime, or bridging one into `ActorSystem` messaging. The runtime counterpart to the **Visual Script Editor** panel (see `skills/visual-script.md`) is split in two: a tiny per-entity `VisualScriptState` component (just a `graphId`) and a `VisualScriptSystem` that compiles the shared graph and runs it. The graph itself is shared, immutable data held in a registry, never copied onto each entity.

---

## Setup

```typescript
import {
  VisualScriptGraphBuilder,
  VisualScriptState,
  VisualScriptSystem,
  registerVisualScriptGraph,
} from '@emptysock/engine'

const b = new VisualScriptGraphBuilder()
const start = b.onUpdate()
const branch = b.branch(/* variableIndex */ 10, 'gt', 2)
const onHigh = b.setVariable(20, 1)
const onLow = b.setVariable(20, 0)

b.connect(start, branch)
 .connect(branch, onHigh, 0)   // true edge
 .connect(branch, onLow, 1)    // false edge

registerVisualScriptGraph('door-logic', b.build())
const door = this.spawn('door')                       // inside a Scene
door.add(VisualScriptState, { graphId: 'door-logic' })

const vs = new VisualScriptSystem({ variables: ctx.variables })   // omit for an isolated VariableStore
```

`getVisualScriptGraph(id)` reads a registered graph and `unregisterVisualScriptGraph(id)` removes it. Re-registering a `graphId` with new data? Call `vs.invalidate(graphId)` so the cached compiled module is rebuilt.

---

## Node types

| Kind | Role | Fields |
|---|---|---|
| `onUpdate` | Entry point, fires on every `VisualScriptSystem.update(scene)` call | none |
| `onEvent` | Entry point, fires when `fireEvent(scene, eventType)` is called | `eventType` |
| `sequence` | Passes execution straight through | none |
| `branch` | Reads a `VariableStore` variable and compares it | `variableIndex`, `comparator` (`'eq'\|'neq'\|'gt'\|'lt'\|'gte'\|'lte'`), `value`; `next[0]` = true edge, `next[1]` = false edge |
| `getVariable` | Reads a `VariableStore` variable into the per-trigger evaluation scope | `variableIndex`, `outputKey` |
| `setVariable` | Writes a literal or a scoped value into `VariableStore` | `variableIndex`, `value: number \| { fromKey: string }` |
| `getSwitch` / `setSwitch` | Same as above for `VariableStore` boolean switches | `switchIndex`, `outputKey` / `value` |
| `sendMessage` | Sends a `Message` through `ActorSystem.send()` | `targetActorId`, `messageType`, `payload?` |

Every node carries a `next: string[]` array of node ids it wires forward to. `branch` is the only node with two ports; every other kind uses `next[0]`.

---

## Driving the graph

```typescript
// Every frame, from a scene's onUpdate (synchronous):
vs.update(this)                    // runs every VisualScriptState entity's onUpdate chains

// From game logic, anywhere you have the scene:
vs.fireEvent(this, 'door_opened')  // runs every matching onEvent chain
const store = vs.variables         // the VariableStore the system is bound to
```

---

## Bridging to the Actor Model

A `sendMessage` node needs an `ActorSystem`, passed to `VisualScriptSystem` at construction (there is no setter):

```typescript
import { ActorSystem, VisualScriptGraphBuilder, VisualScriptSystem } from '@emptysock/engine'

const actors = new ActorSystem()
const b = new VisualScriptGraphBuilder()
const onOpen = b.onEvent('door_opened')
const notify = b.sendMessage('guard-1', 'alert', { reason: 'door' })
b.connect(onOpen, notify)

const vs = new VisualScriptSystem({ actorSystem: actors })
```

---

## Execution semantics

- A trigger fires, and execution walks `next` edges synchronously node-by-node until it hits a node with no outgoing edge for the port it took.
- A per-entity evaluation scope (cleared before each trigger) lets `getVariable`/`getSwitch` results flow into a later `setVariable` via `{ fromKey }`.
- A defensive step cap (10,000 steps per trigger) keeps a graph that wires back into itself from hanging the frame.

---

## Rules

| Wrong | Right |
|---|---|
| Copying the graph JSON onto each entity | Register once with `registerVisualScriptGraph`, reference by `graphId` |
| Wiring `sendMessage` nodes without an `actorSystem` | Pass `actorSystem` to the `VisualScriptSystem` constructor |
| Hand-writing the `VisualScriptGraph` JSON shape from scratch | Use `VisualScriptGraphBuilder` |
| Expecting `getVariable` output to persist across triggers | The evaluation scope resets before each trigger |
| Editing a registered graph and expecting the live system to notice | Re-register, then `vs.invalidate(graphId)` |
