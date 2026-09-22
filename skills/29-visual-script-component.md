# VisualScriptComponent

**Use this when** you're running a visual-script graph at runtime, or bridging one into `ActorSystem` messaging. `VisualScriptComponent` is an ECS `Component` that holds a serialized node graph and interprets it every frame or on a fired event — it's the runtime counterpart to the **Visual Script Editor** panel (see `skills/visual-script.md`). The panel authors a `VisualScriptGraph`; this component walks it against a `VariableStore` and, optionally, an `ActorSystem`.

---

## Setup

```typescript
import { VisualScriptComponent, VisualScriptGraphBuilder } from '@emptysock/engine'

const b = new VisualScriptGraphBuilder()
const start = b.onUpdate()
const branch = b.branch(/* variableIndex */ 10, 'gt', 2)
const onHigh = b.setVariable(20, 1)
const onLow = b.setVariable(20, 0)

b.connect(start, branch)
 .connect(branch, onHigh, 0)   // true edge
 .connect(branch, onLow, 1)    // false edge

const vs = new VisualScriptComponent({ graph: b.build() })
entity.addComponent(vs)
```

`VisualScriptComponent.TYPE` is the string `"VisualScript"` — the key `addComponent`/`getComponent` use, per the engine's component-identity convention (`docs`: component types are identity keys — one `type` string per entity slot).

---

## Node types

| Kind | Role | Fields |
|---|---|---|
| `onUpdate` | Entry point, fires every `update()` call | — |
| `onEvent` | Entry point, fires when `fireEvent(eventType)` is called | `eventType` |
| `sequence` | Passes execution straight through | — |
| `branch` | Reads a `VariableStore` variable and compares it | `variableIndex`, `comparator` (`'eq'\|'neq'\|'gt'\|'lt'\|'gte'\|'lte'`), `value`; `next[0]` = true edge, `next[1]` = false edge |
| `getVariable` | Reads a `VariableStore` variable into the per-tick evaluation scope | `variableIndex`, `outputKey` |
| `setVariable` | Writes a literal or a scoped value into `VariableStore` | `variableIndex`, `value: number \| { fromKey: string }` |
| `getSwitch` / `setSwitch` | Same as above for `VariableStore` boolean switches | `switchIndex`, `outputKey` / `value` |
| `sendMessage` | Sends a real `Message` through `ActorSystem.send()` | `targetActorId`, `messageType`, `payload?` |

Every node carries a `next: string[]` array of node ids it wires forward to (execution-output ports). `branch` is the only node with two ports; every other node kind uses `next[0]`.

---

## Driving the graph

```typescript
// Every frame, from the entity's normal component update pass:
override onUpdate(dt: number): void {
  vs.update(dt)   // runs every onUpdate node's chain once — vs.update is not async
}

// From game logic, anywhere:
vs.fireEvent('door_opened')   // runs every onEvent node whose eventType matches
```

Swap the graph or wire an `ActorSystem` after construction, and read the live store the interpreter is bound to:

```typescript
vs.setGraph(nextGraph)
vs.setActorSystem(actorSystem)
const store = vs.variableStore
```

---

## Bridging to the Actor Model

A `sendMessage` node needs an `ActorSystem` — pass one at construction or wire it later:

```typescript
import { ActorSystem, VisualScriptComponent, VisualScriptGraphBuilder } from '@emptysock/engine'

const actors = new ActorSystem()   // one per scene — create in onLoad, destroy in onDestroy
const b = new VisualScriptGraphBuilder()
const onOpen = b.onEvent('door_opened')
const notify = b.sendMessage('guard-1', 'alert', { reason: 'door' })
b.connect(onOpen, notify)

const vs = new VisualScriptComponent({ graph: b.build(), actorSystem: actors })
```

---

## Execution semantics

- A trigger fires, and execution walks `next` edges synchronously node-by-node until it hits a node with no outgoing edge for the port it took.
- A per-tick evaluation scope (cleared at the start of each trigger) lets `getVariable`/`getSwitch` results flow into a later `setVariable` via `{ fromKey }` — read a variable once, reuse it across several writes in the same trigger.
- A defensive step cap (10,000 steps per trigger) keeps a graph that wires back into itself from hanging the frame. It logs a warning and bails instead of looping forever — the same class of bug as the ActorSystem mailbox-drain case, just wearing a different hat.

---

## Rules

| Wrong | Right |
|---|---|
| Calling `vs.update()` as `async` | The engine calls component `update()` synchronously — reach for coroutines for anything multi-frame |
| Wiring `sendMessage` nodes without passing `actorSystem` | Pass `actorSystem` at construction, or call `setActorSystem()` before the graph runs |
| Hand-writing the `VisualScriptGraph` JSON shape from scratch | Use `VisualScriptGraphBuilder` — it produces exactly the shape the panel round-trips |
| Expecting `getVariable` output to persist across triggers | The evaluation scope resets at the start of each trigger — read it again next time |
