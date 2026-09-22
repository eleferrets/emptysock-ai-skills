# 3D Physics (Rapier3D)

**Use this when** you need real 3D collisions and rigid bodies, not the 2D `PhysicsBody` component. Full Rapier3D integration, loaded via dynamic import so games that never touch 3D don't pay for it. Requires an async `init()` before you can add a single body — Rapier's WASM has to spin up first.

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
- Always `await physics.init()` before adding bodies — Rapier's WASM has to be up and running first.
- Call `physics.destroy()` when the scene unloads. Rapier's memory lives outside the JS heap, so the garbage collector has no idea it exists — skip this and it just piles up.
- Sensor bodies (`isSensor: true`) detect overlaps without pushing anything around — good for trigger volumes.
