# Actor Model & Multiplayer

**Use this when** you're writing game logic that talks to other game logic — enemy AI, NPC coordination, or anything multiplayer — and you want it to survive contact with real bugs. EmptySock uses a mailbox-based Actor Model: actors only ever talk to each other by sending messages, never by reaching into each other's state. No shared mutable state means no "which order did these two things happen in" headaches.

## Core classes

```typescript
import { Actor, ActorSystem, Message, ActorId } from '@emptysock/engine';

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

## Multiplayer with NetworkActor + Transport

NetworkActor is opt-in — nothing forces multiplayer weight onto a single-player game. Wire up a `Transport` (WebSocket, WebRTC, whatever you like) and remote messages land in the same `receive()` method as local ones:

```typescript
import { NetworkActor, Transport, TransportMessage } from '@emptysock/engine';

class RemotePlayerActor extends NetworkActor {
  receive(msg: Message): void { /* handles both local and remote messages */ }
  sendToAll(msg: Message): void {
    this.sendRemote('broadcast', msg);
  }
}

// Plug in any transport without touching actor logic:
class MyWebSocketTransport implements Transport {
  private ws: WebSocket;
  private _handler: ((p: TransportMessage) => void) | null = null;
  constructor(url: string) { this.ws = new WebSocket(url); }
  async connect(): Promise<void> { /* wait for open */ }
  disconnect(): void { this.ws.close(); }
  send(actorId: string, msg: Record<string, unknown>): void {
    this.ws.send(JSON.stringify({ actorId, msg }));
  }
  onReceive(handler: (p: TransportMessage) => void): void {
    this._handler = handler;
    this.ws.onmessage = (e) => handler(JSON.parse(e.data) as TransportMessage);
  }
}

const transport = new MyWebSocketTransport('wss://game.example.com');
await transport.connect();
const remote = new RemotePlayerActor('remote-1');
remote.setTransport(transport);
system.register(remote);
```

## Listing all actors

`getAll()` iterates every actor currently registered in the system — handy for save snapshots, debug UIs, or filtering before a broadcast.

```typescript
const allActors: Actor[] = system.getAll();
for (const actor of allActors) {
  console.log(actor.id);
}
```

## Tips
- One ActorSystem per scene; call `system.update(dt)` in your game loop.
- `broadcast()` is O(n) — prefer targeted `send()` for high-frequency messages.
- Actors are destroyed when `system.unregister(id)` is called — clean up handles in `onStop()`.
- Each actor's inbox tops out at **1,000 messages**. Past that, new messages get dropped with a console warning. Seeing that warning means the actor can't drain its mailbox fast enough — split the work up or send less often.
- Messages sent inside `receive()` are processed in the **same flush pass**, not next frame. If actor A messages B and B messages back to A, both inboxes drain in the same frame — watch out for that kind of mutual back-and-forth, it can spiral.
- `getAll()` hands back a snapshot array. Mutating it does nothing to the real ActorSystem — it's just a look, not a lever.
