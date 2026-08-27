# Actor Model & Multiplayer

EmptySock Engine uses a mailbox-based Actor Model for decoupled game logic. Actors communicate only via messages — no shared mutable state.

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

NetworkActor is an opt-in base class. Wire a `Transport` (WebSocket, WebRTC, etc.) to route remote messages into the same `receive()` method:

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

## Tips
- One ActorSystem per scene; call `system.update(dt)` in your game loop.
- `broadcast()` is O(n) — prefer targeted `send()` for high-frequency messages.
- Actors are destroyed when `system.unregister(id)` is called — clean up handles in `onStop()`.
