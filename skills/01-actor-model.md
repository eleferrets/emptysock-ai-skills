# Actor Model

**Use this when** you're writing game logic that talks to other game logic — enemy AI, NPC coordination, and you want it to survive contact with real bugs. EmptySock uses a mailbox-based Actor Model: actors only ever talk to each other by sending messages, never by reaching into each other's state. No shared mutable state means no "which order did these two things happen in" headaches.

## Core classes

```typescript
import { Actor, ActorSystem, type Message, type ActorId } from '@emptysock/engine';

class PlayerActor extends Actor {
  receive(msg: Message): void {
    if (msg.type === 'MOVE') {
      const { dx, dy } = msg as Message & { dx: number; dy: number };
      // handle movement
    }
  }
  update(dt: number): void { /* called every frame */ }
}

const system = new ActorSystem();
const player = new PlayerActor('player-1');
system.register(player);
system.send('player-1', { type: 'MOVE', dx: 5, dy: 0 });
system.broadcast({ type: 'TICK', dt: 0.016 });
// In game loop:
system.update(dt);
```

## Multiplayer

The core engine has no transport or `NetworkActor` class. Multiplayer lives in the optional `@emptysock/network` package (Colyseus-based); see `skills/35-network-package.md`.

## Listing all actors

`getAll()` iterates every actor currently registered in the system — handy for save snapshots, debug UIs, or filtering before a broadcast.

```typescript
const allActors: Actor[] = system.getAll();
for (const actor of allActors) {
  console.log(actor.id);
}
```

## Tips
- `Game` creates one ActorSystem per scene (`ctx.actors`) and updates it for you; only construct your own for standalone use, then call `system.update(dt)` yourself.
- `broadcast()` is O(n) — prefer targeted `send()` for high-frequency messages.
- `system.unregister(id)` removes an actor; clean up in the actor's `onStop()` / `destroy()` overrides. Actors also have `start()`, `stop()`, `isRunning`, `inboxSize` and `send(msg)`.
- Each actor's inbox tops out at **1,000 messages**. Past that, new messages get dropped with a console warning. Seeing that warning means the actor can't drain its mailbox fast enough — split the work up or send less often.
- Messages sent inside `receive()` are processed in the **same flush pass**, not next frame. If actor A messages B and B messages back to A, both inboxes drain in the same frame — watch out for that kind of mutual back-and-forth, it can spiral.
- `getAll()` hands back a snapshot array. Mutating it does nothing to the real ActorSystem — it's just a look, not a lever.
