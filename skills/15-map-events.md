# MapEventSystem

`MapEventSystem` handles RPG Maker MV-style tile-bound events on a Tilemap. Each event occupies one tile and carries a trigger type and a list of commands. The system drives command execution sequentially, awaiting async commands before moving to the next.

---

## Quick start

```typescript
import { MapEventSystem, variableStore, VNSystem } from '@emptysock/engine'

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
    await vn.showDialogue(cmd.speaker, cmd.text)
  } else if (cmd.type === 'set-variable') {
    variableStore.setVar(cmd.index, cmd.value)
    variableStore.save()
  } else if (cmd.type === 'set-switch') {
    variableStore.setSwitch(cmd.index, cmd.value)
    variableStore.save()
  } else if (cmd.type === 'transition-scene') {
    SceneManager.loadScene(cmd.scene)
  }
})

// In onUpdate — pass player tile coordinates and whether action was pressed this frame
events.update(playerTileX, playerTileY, input.isJustPressed('KeyZ'))
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
events.toJSON(): MapEvent[]
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
}
```

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

```typescript
// Serialise (include in save slot)
const saved = events.toJSON()

// Restore
events.loadEvents(saved)
```

---

## Rules

- Call `setHandler` before any `update` call.
- `autorun` and `player-touch` events run at most once per `loadEvents` call — they do not re-fire until `loadEvents` is called again (i.e., on a new map load).
- `action-button` events can be re-triggered once their command list finishes.
- `parallel` events re-trigger every frame — keep their command lists short (one command) or they will queue faster than they execute.
- The system runs one event at a time. A second event will not start until the current one finishes.
