# CGGallery

**Use this when** you're building the "unlockables" screen for a visual novel's CG art. `CGGallery` tracks which full-screen images the player has seen, persists it through `SaveSystem`, and gives you counts and entries to build a gallery UI from. Unlocking happens automatically from Story Graph nodes, or you can call it manually.

## Import

```typescript
import { CGGallery } from '@emptysock/engine'
```

## Quick start

```typescript
import { CGGallery, SaveSystem } from '@emptysock/engine'

// In onLoad — define all gallery entries:
const gallery = new CGGallery({
  entries: [
    { id: 'cg-01', imagePath: 'assets/cg/forest_encounter.jpg', title: 'Forest Encounter' },
    { id: 'cg-02', imagePath: 'assets/cg/castle_throne.jpg',    title: 'Throne Room'      },
    { id: 'cg-03', imagePath: 'assets/cg/ending_a.jpg',         title: 'Ending A'         },
  ],
})

// Load previously unlocked state:
gallery.load()

// Unlock a CG (e.g., when a scene plays it):
gallery.unlock('cg-01')

// Check unlock state:
if (gallery.isUnlocked('cg-01')) {
  showCGThumbnail('cg-01')
}

// Build gallery screen UI:
const unlocked = gallery.unlockedEntries  // CGEntry[]
const total    = gallery.totalCount       // 3
const count    = gallery.unlockedCount    // how many are unlocked
```

## Unlocking from Story Graph nodes

Pass the CG id from your `VNSystem` node callback when a CG is shown:

```typescript
import { CGGallery } from '@emptysock/engine'
import { VNSystem, VNBackgroundLayer } from '@emptysock/vn'

vn.onNode((node) => {
  if (node.type === 'dialogue' && node.cgPath !== undefined) {
    gallery.unlock(node.cgPath)   // marks as unlocked and persists via SaveSystem
    bg.showCG(node.cgPath)
  }
})
```

## API reference

| Method / Property | Signature | Description |
|-------------------|-----------|-------------|
| `gallery.load` | `(): void` | Restore unlock state from `SaveSystem`. Call once in `onLoad`. |
| `gallery.unlock` | `(id: string): void` | Mark a CG as unlocked and persist via `SaveSystem`. |
| `gallery.unlockFromNode` | `(cgId: string): void` | Same as `unlock` — named variant for Story Graph node integration. |
| `gallery.isUnlocked` | `(id: string): boolean` | True if the CG with this id has been unlocked. |
| `gallery.entries` | `ReadonlyArray<CGEntry>` | All registered entries (locked and unlocked). |
| `gallery.unlockedEntries` | `CGEntry[]` | Entries the player has seen. |
| `gallery.totalCount` | `number` | Total number of registered entries. |
| `gallery.unlockedCount` | `number` | Number of unlocked entries. |

## Constructor options

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `entries` | `CGEntry[]` | required | All gallery entries with `id`, `imagePath`, and optional `title`. |
| `saveSystem` | `typeof SaveSystem` | `SaveSystem` | Override for testing. |
| `saveSlot` | `string` | `'cg-gallery'` | Save slot name used for persistence. |

## Notes

- Call `gallery.load()` in `onLoad` before checking any unlock state.
- `unlock()` and `unlockFromNode()` write straight to `SaveSystem` — no manual save call needed.
- `CGGallery` doesn't render anything itself. Use `VNBackgroundLayer.showCG()` to actually show an image.
- Keep one `CGGallery` instance alive for the whole app's lifetime — a module-level variable or a long-lived scene works.
