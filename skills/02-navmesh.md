# PathfindingSystem

Grid-based A* pathfinding and scene NavMesh construction. All methods are static — no instantiation needed.

## Import

```typescript
import { PathfindingSystem } from '@emptysock/engine'
```

## Grid pathfinding (most common)

Load a tilemap layer as a grid and call `PathfindingSystem.findPath()`:

```typescript
import { TilemapSystem, PathfindingSystem } from '@emptysock/engine'

// In onLoad:
const map  = TilemapSystem.load('level1.esmap')
const grid = map.getLayer('Collision').asGrid()   // Grid from tile layer

// In a coroutine or onLoad — findPath is async:
const path = await PathfindingSystem.findPath({
  from:          { x: playerTileX, y: playerTileY },
  to:            { x: targetTileX, y: targetTileY },
  grid,
  allowDiagonal: false,
})
// path is a Path (array of { x, y } waypoints); empty if no path exists
```

## NavMesh from a scene

For scene-level polygon pathfinding, build a NavMesh from the loaded scene:

```typescript
// In onLoad — after the scene is set up:
const navMesh = PathfindingSystem.buildNavMesh(this)
// navMesh is a NavMesh object used with PathFollower component
```

## Debug overlay

```typescript
// Toggle the debug path-draw overlay (disable in release builds):
PathfindingSystem.debugDraw(true)
```

## API reference

| Method | Signature | Notes |
|--------|-----------|-------|
| `PathfindingSystem.findPath` | `(opts: { from: Point, to: Point, grid: Grid, allowDiagonal?: boolean }): Promise<Path>` | Grid A*. Returns empty path if unreachable. |
| `PathfindingSystem.buildNavMesh` | `(scene: Scene): NavMesh` | Build a NavMesh from the current scene's geometry. Build offline and cache — do not call every frame. |
| `PathfindingSystem.debugDraw` | `(enabled: boolean): void` | Toggle visual path overlay. |

## Notes

- `findPath` returns a `Promise` — call it inside a coroutine (`function*`) or in `onLoad`; never inside `onUpdate`.
- Build the NavMesh in `onLoad`, not on demand. NavMesh construction is O(n log n) over scene geometry.
- Attach a `PathFollower` component to an entity to drive it along the returned `Path` automatically.
- `PathfindingSystem` has no instance state — it is a static utility class, not a singleton or per-scene object.

---

# NavMeshSystem

`NavMeshSystem` performs polygon-graph A* on a pre-built convex polygon dataset. Use it for open-world navigation where grid cells are too coarse. Unlike `PathfindingSystem`, it accepts pre-baked data rather than generating the mesh at runtime.

## Import

```typescript
import { NavMeshSystem, type NavMeshData, type Vec2 } from '@emptysock/engine'
```

## Setup

```typescript
import navData from './assets/maps/world.navmesh.json'

// In onLoad:
const nav = new NavMeshSystem()
nav.load(navData as NavMeshData)   // load pre-built polygon graph

// Find a path — synchronous, no await needed:
const waypoints: Vec2[] = nav.findPath(
  { x: player.position.x, y: player.position.y },
  { x: target.position.x, y: target.position.y },
)
// Empty array if no path exists.
```

## NavMeshData shape

```typescript
// NavMeshData is a plain JSON object — build offline in the level editor:
{
  "polygons": [
    {
      "id": 0,
      "vertices": [{ "x": 0, "y": 0 }, { "x": 64, "y": 0 }, { "x": 64, "y": 64 }],
      "centroid": { "x": 42, "y": 21 },
      "neighbours": [1, 3]
    }
    // … more convex polygons
  ]
}
```

## API reference

| Method | Signature | Notes |
|--------|-----------|-------|
| `load` | `(data: NavMeshData): void` | Load the polygon graph. Call once in onLoad. |
| `findPath` | `(from: Vec2, to: Vec2): Vec2[]` | Synchronous A* on polygon graph. Returns [] if no path. |

## When to use which

| System | Use when |
|--------|---------|
| `PathfindingSystem` | Tile-based levels with a collision grid |
| `NavMeshSystem` | Open-world or irregular geometry; navmesh built offline |
