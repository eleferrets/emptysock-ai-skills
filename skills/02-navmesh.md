# NavMesh Pathfinding

Polygon-based 2D pathfinding using A* on a convex polygon graph.

## Usage

```typescript
import { NavMeshSystem, NavMeshData } from '@emptysock/engine';

const navMesh = new NavMeshSystem();

const data: NavMeshData = {
  polygons: [
    {
      id: 0,
      vertices: [{ x: 0, y: 0 }, { x: 100, y: 0 }, { x: 100, y: 100 }, { x: 0, y: 100 }],
      centroid: { x: 50, y: 50 },
      neighbours: [1],
    },
    {
      id: 1,
      vertices: [{ x: 100, y: 0 }, { x: 200, y: 0 }, { x: 200, y: 100 }, { x: 100, y: 100 }],
      centroid: { x: 150, y: 50 },
      neighbours: [0],
    },
  ],
};

navMesh.load(data);
const path = navMesh.findPath({ x: 10, y: 10 }, { x: 190, y: 90 });
// path is an array of Vec2 waypoints (centroids), empty if no path exists
```

## Notes
- Build navmesh data offline (e.g., with a tilemap editor) and load at runtime.
- `findPath` uses point-in-polygon containment first, then centroid distance fallback.
- Call `navMesh.update(dt)` in your game loop (no-op currently but reserved for dynamic obstacles).
