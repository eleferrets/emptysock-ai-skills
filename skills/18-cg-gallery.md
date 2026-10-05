# CGGallery

**Use this when** you're building the "unlockables" screen for a visual novel's CG art. `CGGallery` tracks which full-screen images the player has seen, persists the unlock flags through a `StorageAdapter`, and gives you counts and entries to build a gallery UI from.

## Import

```typescript
import { CGGallery, MemoryStorageAdapter, type StorageAdapter } from '@emptysock/engine'
```

## Quick start

```typescript
const gallery = new CGGallery({
  entries: [
    { id: 'cg-01', imagePath: 'assets/cg/forest_encounter.jpg', title: 'Forest Encounter' },
    { id: 'cg-02', imagePath: 'assets/cg/castle_throne.jpg',    title: 'Throne Room'      },
    { id: 'cg-03', imagePath: 'assets/cg/ending_a.jpg',         title: 'Ending A'         },
  ],
})

// Persistence goes through a StorageAdapter you provide (async):
const adapter: StorageAdapter = myAdapter            // e.g. new MemoryStorageAdapter() in tests
await gallery.load(adapter)                          // restore previous unlocks

await gallery.unlock(adapter, 'cg-01')               // marks as unlocked and persists immediately

if (gallery.isUnlocked('cg-01')) showCGThumbnail('cg-01')

const unlocked = gallery.unlockedEntries             // CGEntry[]
const total    = gallery.totalCount                  // 3
const count    = gallery.unlockedCount
```

## Unlocking from Story Graph nodes

`VNSystem` reports nodes that carry a `cgPath` through the listener's `onCGNode` callback (see `skills/08-story-graph.md`). Match the path to a gallery entry id (or look up the entry by `imagePath`):

```typescript
vn.setListener({
  onCGNode(cgPath) {
    const entry = gallery.entries.find((e) => e.imagePath === cgPath)
    if (entry !== undefined) void gallery.unlock(adapter, entry.id)
    bg.showCG(cgPath)               // VNBackgroundLayer, see skills/12-vn-textbox.md
  },
})
```

## API reference

| Member | Signature | Description |
|--------|-----------|-------------|
| `load` | `(adapter: StorageAdapter, key = 'emptysock_cg_gallery'): Promise<void>` | Restore unlock flags; leaves state untouched if nothing stored or malformed |
| `unlock` | `(adapter: StorageAdapter, id: string, key = 'emptysock_cg_gallery'): Promise<void>` | Unlock and persist; no-op if already unlocked |
| `isUnlocked` | `(id: string): boolean` | Unlock state |
| `entries` | `ReadonlyArray<CGEntry>` | All entries |
| `unlockedEntries` | `CGEntry[]` | Entries the player has seen |
| `totalCount` | `number` | Number of entries |
| `unlockedCount` | `number` | Number of unlocked ids |

`CGEntry` is `{ id: string; imagePath: string; title?: string }`. The constructor takes `{ entries }` only.

## Notes

- `load` and `unlock` are async; await them.
- Persistence is a JSON object `{ [id]: true }` under the storage key. It does not go through `SaveSystem`.
- `CGGallery` renders nothing. Use `VNBackgroundLayer.showCG()` to display an image.
- Keep one `CGGallery` alive for the whole app lifetime (module-level variable or a service on `game.services`).
