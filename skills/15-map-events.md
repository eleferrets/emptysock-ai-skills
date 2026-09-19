# MapEventSystem

`MapEventSystem` handles RPG Maker MV-style tile-bound events on a Tilemap. Each event occupies one tile and carries a trigger type and a list of commands. The system drives command execution sequentially, awaiting async commands before moving to the next.

---

## Quick start

```typescript
import { MapEventSystem, variableStore, VNSystem, SceneManager, InputSystem } from '@emptysock/engine'

// Defaults to the shared `variableStore` singleton — pass a different
// `VariableStore` instance to isolate an event system's `when` gates.
const events = new MapEventSystem()
const vn = new VNSystem()

// Define events
events.addEvent({
  id: 'chest-01',
  tileX: 5, tileY: 3,
  trigger: 'action-button',
  commands: [
    { type: 'show-dialogue', speaker: 'Chest', text: 'You found 100 gold!' },
    { type: 'set-variable', index: 1, value: variableStore.getVar(1) + 100 },
    { type: 'set-switch', index: 2, value: true },
  ],
})

events.addEvent({
  id: 'intro-cutscene',
  tileX: 0, tileY: 0,
  trigger: 'autorun',
  commands: [
    { type: 'show-dialogue', speaker: 'Narrator', text: 'Your journey begins…' },
    { type: 'transition-scene', scene: 'MapScene' },
  ],
})

// Wire the command handler
events.setHandler(async (cmd) => {
  if (cmd.type === 'show-dialogue') {
    // Load a single-node dialogue tree and display it:
    const off = vn.onNode((node) => { /* render node.speaker, node.text */ })
    vn.load({ nodes: { n0: { type: 'dialogue', speaker: cmd.speaker, text: cmd.text } }, startNode: 'n0' })
    await new Promise<void>((resolve) => { const u = vn.onEnd(() => { u(); off(); resolve() }) })
  } else if (cmd.type === 'set-variable') {
    variableStore.setVar(cmd.index, cmd.value)
    variableStore.save()
  } else if (cmd.type === 'set-switch') {
    variableStore.setSwitch(cmd.index, cmd.value)
    variableStore.save()
  } else if (cmd.type === 'transition-scene') {
    SceneManager.load(cmd.scene)
  }
})

// In onUpdate — pass player tile coordinates and whether action was pressed this frame
// (assuming this.input is an InputSystem instance, flushed at the start of onUpdate)
events.update(playerTileX, playerTileY, this.input.isKeyPressed('KeyZ'))
```

---

## API

```typescript
new MapEventSystem()

events.addEvent(event: MapEvent): void
events.removeEvent(id: string): void
events.loadEvents(events: MapEvent[]): void
events.setHandler(handler: EventCommandHandler): void
events.update(playerTileX: number, playerTileY: number, actionPressed: boolean): void
```

---

## MapEvent

```typescript
interface MapEvent {
  id: string
  tileX: number
  tileY: number
  trigger: EventTriggerType
  commands: EventCommand[]
  /**
   * Optional gate evaluated against the VariableStore before the event is
   * allowed to run. When present and false, update() skips the event
   * entirely — it never triggers, autoruns, or fires as a parallel process —
   * and it is re-checked every frame, so the event starts working the moment
   * the condition becomes true.
   */
  when?: VariableCondition
}
```

### Gating an event on a variable

```typescript
import { variableStore, type MapEvent } from '@emptysock/engine'

// Invisible (never triggers) until switch 10 (set elsewhere, e.g. by another
// event or a Story Graph node) flips to true:
const throneRoomDoor: MapEvent = {
  id: 'throne-room-door',
  tileX: 0, tileY: 0,
  trigger: 'autorun',
  when: { kind: 'switch', index: 10, equals: true },
  commands: [{ type: 'show-dialogue', speaker: 'Guard', text: 'The way is open.' }],
}
events.addEvent(throneRoomDoor)
```

See `skills/13-variable-store.md` for the full `VariableCondition` shape and `evaluateCondition()`.

### Trigger types

| Trigger | When it fires |
|---------|--------------|
| `'autorun'` | Immediately when the player enters the map (fires once) |
| `'player-touch'` | When the player steps onto the event tile (fires once per map entry) |
| `'action-button'` | When the player is on the tile and presses the action key |
| `'parallel'` | Every frame while the map is active |

### Event commands

```typescript
type EventCommand =
  | { type: 'show-dialogue'; speaker: string; text: string }
  | { type: 'set-variable'; index: number; value: number }
  | { type: 'set-switch'; index: number; value: boolean }
  | { type: 'play-audio'; src: string; volume?: number }
  | { type: 'transition-scene'; scene: string; transition?: string }
  | { type: 'move-character'; entityId: string; tileX: number; tileY: number }
```

### EventCommandHandler

```typescript
type EventCommandHandler = (cmd: EventCommand) => void | Promise<void>
```

The handler may be async. The system awaits it before executing the next command.

---

## Persisting events

To save and restore map event state across sessions, snapshot the events list yourself and pass it back to `loadEvents()`:

```typescript
import { SaveSystem } from '@emptysock/engine'

// Capture the current event definitions (they are plain data — no toJSON needed):
const saved = myEventDefinitions   // the same MapEvent[] you passed to loadEvents()

const saves = new SaveSystem()

// Include in a save slot:
saves.save('slot-1', { scene: 'Map01', data: { events: saved } })

// Restore on load — load() already validates against GameSaveSlot:
const slot = saves.load('slot-1')
if (slot !== null) {
  events.loadEvents(slot.data.events as MapEvent[])
}
```

---

## Rules

- Call `setHandler` before any `update` call.
- `autorun` and `player-touch` events run at most once per `loadEvents` call — they do not re-fire until `loadEvents` is called again (i.e., on a new map load).
- `action-button` events can be re-triggered once their command list finishes.
- `parallel` events re-trigger every frame — keep their command lists short (one command) or they will queue faster than they execute.
- The system runs one event at a time. A second event will not start until the current one finishes.
