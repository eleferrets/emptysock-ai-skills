# AStarSearch, Tilemap grids, and NavMeshSystem

**Use this when** you need enemy/NPC pathfinding: grid-based A* over a tile grid, or a polygon navmesh for open terrain.

- `AStarSearch` is a generic, dependency-free A* function exported from `@emptysock/engine`. You supply the graph (neighbours, heuristic, key).
- `Tilemap`, `TilemapSystem`, `NavMeshSystem`, and `AutoTileSystem` live in the optional `@emptysock/tilemap` package.
- There is no `PathfindingSystem`, no `PathFollower` component, and no runtime navmesh builder. Build navmesh data offline and ship it as JSON.

## Generic A*: `AStarSearch`

```typescript
import { AStarSearch } from '@emptysock/engine'
import { Tilemap, TilemapSystem } from '@emptysock/tilemap'

interface Cell { x: number; y: number }

const map: Tilemap | undefined = TilemapSystem.get('level1')   // registered earlier with TilemapSystem.register(data)
const grid = map?.asGrid() ?? []        // boolean[][] indexed [row][col]; true = walkable, false = a solid tile

const result = AStarSearch<Cell>({
  start: { x: 1, y: 1 },
  isGoal: (n) => n.x === 8 && n.y === 5,
  key: (n) => n.y * 1000 + n.x,
  heuristic: (n) => Math.abs(n.x - 8) + Math.abs(n.y - 5),
  neighbours: (n) => {
    const out: { node: Cell; cost: number }[] = []
    for (const [dx, dy] of [[1, 0], [-1, 0], [0, 1], [0, -1]] as const) {
      const x = n.x + dx
      const y = n.y + dy
      if (grid[y]?.[x] === true) out.push({ node: { x, y }, cost: 1 })
    }
    return out
  },
})
// result.found: boolean, result.path: ReadonlyArray<Cell> (start to goal)
```

`AStarSearch` is synchronous. Run it from `onLoad`, a coroutine, or a throttled update, not every frame for many agents.

## Tilemap data and grids

| API | Notes |
|-----|-------|
| `TilemapSystem.register(data: TilemapData): Tilemap` | Register a map by `data.name` |
| `TilemapSystem.loadInto(scene, name): Tilemap` | Spawns a `Transform` entity named after the map and binds it |
| `TilemapSystem.get(name)` / `.remove(name)` | Lookup / unregister |
| `tilemap.getLayer(name)` | `TilemapLayer \| undefined` (`name`, `cells: TileCell[][]`, `visible`, `opacity`) |
| `tilemap.asGrid()` | `boolean[][]` walkability grid (`true` = free, `false` where any layer cell has `solid: true`) |
| `tilemap.tileAt(worldX, worldY, layerName)` | `TileCell \| null` (`tileIndex`, `solid?`) |
| `tilemap.width` / `.height` / `.entity` | Pixel size and bound entity |

---

# NavMeshSystem

`NavMeshSystem` runs polygon-graph A* on pre-built convex polygon data. Use it where grid cells are too coarse.

```typescript
import { NavMeshSystem, type NavMeshData } from '@emptysock/tilemap'
import type { Vec2 } from '@emptysock/engine'

const nav = new NavMeshSystem()
nav.load(navData as NavMeshData)                 // call once in onLoad

const waypoints: Vec2[] = nav.findPath(from, to) // synchronous; [] if no path
const snapped: Vec2 | null = nav.nearestNode(point)
```

`NavMeshData` is `{ polygons: NavPolygon[] }` with `NavPolygon = { id, vertices: Vec2[], centroid: Vec2, neighbours: number[] }` (neighbour polygon ids).

| Method | Signature |
|--------|-----------|
| `load` | `(data: NavMeshData): void` |
| `findPath` | `(from: Vec2, to: Vec2): Vec2[]` |
| `nearestNode` | `(point: Vec2): Vec2 \| null` |
| `update` | `(dt: number): void` (no-op; present for system-shape uniformity) |

## When to use which

| Tool | Use when |
|------|----------|
| `AStarSearch` + `Tilemap.asGrid()` | Tile-based levels with a collision grid |
| `NavMeshSystem` | Open or irregular geometry; navmesh built offline |
