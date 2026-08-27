# 3D Physics (Rapier3D)

Full Rapier3D integration via `@dimforge/rapier3d-compat`. Dynamic import + async init required.

## Usage

```typescript
import { PhysicsSystem3D } from '@emptysock/engine';

const physics = new PhysicsSystem3D();
await physics.init({ x: 0, y: -9.81, z: 0 }); // must await

// Add a dynamic box
const box = physics.addBody({
  bodyType: 'dynamic',
  shape: 'box',
  halfExtents: { x: 0.5, y: 0.5, z: 0.5 },
  position: { x: 0, y: 5, z: 0 },
  density: 1,
  restitution: 0.3,
});

// Add a static floor
physics.addBody({
  bodyType: 'static',
  shape: 'box',
  halfExtents: { x: 50, y: 0.1, z: 50 },
  position: { x: 0, y: 0, z: 0 },
});

// In game loop:
physics.update(dt);

const pos = box.getPosition(); // { x, y, z }
const rot = box.getRotation(); // { x, y, z, w }
box.applyImpulse({ x: 0, y: 10, z: 0 });

physics.removeBody(box.bodyIndex);
physics.destroy(); // free WASM resources on scene unload
```

## Shapes
`box` — `halfExtents: Vec3`  
`sphere` — `radius: number`  
`capsule` — `radius`, `halfHeight`  
`cylinder` — `radius`, `halfHeight`  
`cone` — `radius`, `halfHeight`  

## Body types
`dynamic` — fully simulated  
`static` — immovable collider  
`kinematic` — moved manually, pushes dynamics

## Notes
- Always `await physics.init()` before adding bodies — Rapier WASM must initialize.
- `physics.destroy()` must be called when the scene unloads to free WASM memory.
- Sensor bodies (`isSensor: true`) detect overlaps without generating forces.
