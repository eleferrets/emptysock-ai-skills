# SignalBus

**Use this when** code that has no reference to another system still needs to tell the game something happened: a player died, a door opened, a setting changed. `SignalBus` is a game-wide bus of named signals. There is one per `Game` (`game.signals`, and `ctx.signals` inside a scene), never a bare importable singleton.

It is not `ActorSystem` (addressed mailboxes drained before `update()`; see `skills/01-actor-model.md`). Reach for actors when one specific thing should receive a message, and for the bus when anyone interested may listen.

---

## Basic use

```typescript
import { defineScene, type SignalGroup } from '@emptysock/engine'

let signals: SignalGroup | undefined

export const Level1 = defineScene({
  onLoad(_scene, ctx) {
    signals = ctx.signals.group()                    // subscribe through a group
    signals.on<{ id: number }>('player-died', (p) => respawn(p.id))
    signals.once('level-clear', () => goToNextLevel())
  },
  onUnload() {
    signals?.dispose()                               // removes everything this scene subscribed
  },
})

// Anywhere that has the Game or a scene context:
game.signals.emit('player-died', { id: 3 })          // returns how many listeners ran
```

**Always subscribe through `bus.group()` in `onLoad` and `dispose()` it in `onUnload`.** A handler that outlives its scene runs against destroyed entities in the next one.

---

## API

| Member | Notes |
|---|---|
| `on(name, fn)` / `once(name, fn)` | Subscribe; both return an unsubscribe function. Listeners receive `(payload, name)`. |
| `off(name, fn)` | Remove one listener. |
| `onAny(fn)` | Wildcard tap: called for every emitted signal, after that signal's own listeners. Useful for debug tooling. |
| `emit(name, payload?)` | Calls every listener of `name`, then the wildcards. Returns how many ran. |
| `broadcast(payload?)` | Emits `payload` to every signal name that currently has at least one listener. |
| `listenerCount(name?)` | Listeners for one name, or everything (including wildcards) when omitted. |
| `clear()` | Drop every subscription. |
| `group()` | A `SignalGroup` with `on` / `once` / `emit` / `dispose`. |

## Behaviour that matters

- **Synchronous, in registration order.** `emit` returns after every listener ran.
- **Nested emits run immediately.** A listener that re-emits its own signal loops forever. Don't.
- **One throwing listener never blocks the rest.** Every error is collected and re-thrown after all listeners ran, as a single `AggregateError`.
- **Typed names.** Augment `GameSignals` once and `on`/`emit` type-check:

```typescript
declare module '@emptysock/engine' {
  interface GameSignals {
    'player-died': { id: number }
  }
}
```

## Rules

| Wrong | Right |
|---|---|
| `ctx.signals.on(...)` in `onLoad` with no matching cleanup | `const g = ctx.signals.group()`, then `g.dispose()` in `onUnload` |
| Using a signal to reach one known entity | Send that actor a message, or call it directly |
| Emitting from inside a listener of the same signal | Set a flag and emit once from `onUpdate` |
| `setTimeout(() => emit(...))` for a delayed signal | Start a coroutine (`entity.startCoroutine`) and emit from it |
