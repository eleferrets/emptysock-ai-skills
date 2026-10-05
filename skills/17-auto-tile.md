# AutoTileSystem

**Use this when** you're painting tilemap terrain and want transition tiles (grass-to-dirt edges, wall corners) chosen automatically instead of hand-placed.

`AutoTileSystem` lives in the optional `@emptysock/tilemap` package (not `@emptysock/engine`) and is a normal class: create an instance. It picks tile variants from an 8-neighbour bitmask. Define a rule set per base tile, then call `resolve()` or `applyToLayer()`.

## Import

```typescript
import { AutoTileSystem, type AutoTileRuleSet } from '@emptysock/tilemap'
```

## Quick start

```typescript
const auto = new AutoTileSystem()

auto.addRuleSet({
  id: 'grass',
  baseTileIndex: 1,
  defaultTileIndex: 1,            // used when no rule matches
  rules: [
    // mask bits: 0 NW, 1 N, 2 NE, 3 W, 4 E, 5 SW, 6 S, 7 SE (1 = neighbour is the same terrain)
    // A rule matches when (neighbourMask & rule.mask) === rule.mask; first match wins,
    // so list the most specific masks first.
    { mask: 0b11111111, tileIndex: 10 },   // fully surrounded
    { mask: 0b01000000, tileIndex: 11 },   // south neighbour (at least)
    { mask: 0b00000000, tileIndex: 12 },   // matches anything, so keep it last
  ],
})

// Resolve one cell. tileAt returns the tile index at (col, row); use -1 for empty / out of bounds.
const tileAt = (c: number, r: number): number => grid[r]?.[c] ?? -1
const resolved = auto.resolve(col, row, 1, tileAt)
```

Or resolve a whole sparse layer in place: `auto.applyToLayer(data, baseTileIndex)` where `data` is a `Record<string, number>` keyed `"col,row"` mapping to tile index; every cell holding the base tile (or one of its rule tiles) is re-resolved.

A neighbour counts as "same terrain" when its index equals `baseTileIndex` or is one of that rule set's `tileIndex` values.

## API reference

| Method | Signature |
|--------|-----------|
| `addRuleSet` | `(ruleSet: AutoTileRuleSet): void` (replaces any set with the same `baseTileIndex`) |
| `removeRuleSet` | `(baseTileIndex: number): void` |
| `getRuleSet` | `(baseTileIndex: number): AutoTileRuleSet \| undefined` |
| `resolve` | `(col, row, baseTileIndex, tileAt): number` (returns `baseTileIndex` if no rule set is registered) |
| `applyToLayer` | `(data: Record<string, number>, baseTileIndex: number): void` |
| `toJSON` / `fromJSON` | `(): AutoTileRuleSet[]` / `(ruleSets: AutoTileRuleSet[]): void` |

Types: `AutoTileRule = { mask, tileIndex }`, `AutoTileRuleSet = { id, baseTileIndex, rules, defaultTileIndex }`.

## Notes

- Run `resolve()`/`applyToLayer()` when painting or loading a layer, not every frame.
- `Tilemap` cells are `TileCell[][]` (`tileIndex`, `solid?`); build a `tileAt` closure over `layer.cells[row]?.[col]?.tileIndex`.
