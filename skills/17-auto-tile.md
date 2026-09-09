# AutoTileSystem

`AutoTileSystem` selects tile variants automatically based on an 8-neighbour bitmask, eliminating hand-placed transition tiles. Define rule sets that map neighbour patterns to tile indices; call `resolve()` when painting or regenerating a layer.

## Import

```typescript
import { AutoTileSystem } from '@emptysock/engine'
```

## Quick start

```typescript
import { AutoTileSystem, TilemapSystem } from '@emptysock/engine'

// In onLoad — define a rule set for tile index 1 (grass):
AutoTileSystem.addRuleSet({
  baseTileIndex: 1,
  rules: [
    // Each rule maps an 8-neighbour bitmask (0b76543210, N NE E SE S SW W NW)
    // to the tile index to paint. Add as many variants as your tileset has.
    { mask: 0b00000000, tileIndex: 10 },   // isolated
    { mask: 0b00010000, tileIndex: 11 },   // south neighbour only
    // ... more rules
  ],
})

// Resolve a tile on a layer:
const map   = TilemapSystem.load('level1.esmap')
const layer = map.getLayer('Ground')

// For each tile you want to resolve:
const resolvedIndex = AutoTileSystem.resolve(
  col,
  row,
  1,                          // baseTileIndex
  (c, r) => layer.getTile(c, r)?.index ?? 0
)
// Paint resolvedIndex at (col, row) on the layer.
```

## API reference

| Method | Signature | Description |
|--------|-----------|-------------|
| `AutoTileSystem.addRuleSet` | `(ruleSet: AutoTileRuleSet): void` | Register an auto-tile rule set for a base tile index. Replaces any existing rule set for that index. |
| `AutoTileSystem.removeRuleSet` | `(baseTileIndex: number): void` | Remove the rule set for the given base tile index. |
| `AutoTileSystem.getRuleSet` | `(baseTileIndex: number): AutoTileRuleSet \| undefined` | Retrieve the registered rule set, or `undefined` if none. |
| `AutoTileSystem.resolve` | `(col: number, row: number, baseTileIndex: number, tileAt: (col: number, row: number) => number): number` | Return the tile index to paint at `(col, row)` given neighbour occupancy from `tileAt`. |

## Notes

- `AutoTileSystem` is a static utility — no instantiation required.
- `tileAt(col, row)` should return `0` (or any falsy tile index) for empty or out-of-bounds cells.
- Call `resolve()` offline (in a preprocessing step or in `onLoad`) — do not call it every frame.
- Pair with `TilemapLayer.getTile()` to read neighbour indices and paint the result back via tilemap editor tooling or a custom paint loop.
