# LayerSystem

`LayerSystem` manages named render layers, controls draw order, and applies per-layer camera parallax. Entities are assigned to a layer; `RenderSystem` draws layers in ascending `zOrder`.

---

## Setup (in onLoad)

```typescript
import { LayerSystem } from '@emptysock/engine';

const layers = new LayerSystem();

layers.defineLayer({ name: 'Background', zOrder: 0,  parallax: { x: 0.2, y: 0.2 } });
layers.defineLayer({ name: 'Midground',  zOrder: 10, parallax: { x: 0.6, y: 0.6 } });
layers.defineLayer({ name: 'Gameplay',   zOrder: 20 });                 // scrolls 1:1
layers.defineLayer({ name: 'FX',         zOrder: 30, blendMode: 'additive' });
layers.defineLayer({ name: 'UI',         zOrder: 40, fixed: true });    // camera-fixed

// Register with the scene so RenderSystem picks it up:
scene.setLayerSystem(layers);
```

---

## Assigning entities

```typescript
layers.addToLayer('Background', skyEntity);
layers.addToLayer('Gameplay',   player);
layers.addToLayer('FX',         explosionEmitter);
layers.removeFromLayer('Gameplay', player); // entity persists; just excluded from this layer
```

---

## Visibility and parallax at runtime

```typescript
layers.setVisible('FX', false);          // cull the whole layer from the render pass
layers.setVisible('FX', true);
layers.setParallax('Background', { x: 0.3, y: 0.1 });
```

---

## Introspection

```typescript
for (const layer of layers.sorted()) {   // in ascending zOrder
  console.log(layer.name, layer.zOrder, layer.visible);
}
```

---

## Teardown (in onDestroy)

```typescript
layers.destroy();
```

---

## defineLayer options

| Option | Type | Description |
|--------|------|-------------|
| `name` | `string` | Unique layer identifier |
| `zOrder` | `number` | Ascending draw order — lower renders further back |
| `parallax` | `{ x, y }` | Camera offset multiplier; default `{ x: 1, y: 1 }` |
| `blendMode` | `'normal' \| 'additive'` | WebGL composite mode |
| `fixed` | `boolean` | If true, ignores camera translation (use for UI layers) |

---

## Integration with RenderSystem

Call `scene.setLayerSystem(layers)` once in `onLoad`. The render pipeline then reads layer assignments automatically each frame. Without this call, all entities render in insertion order with no parallax.

---

## Tips

- Use `zOrder` gaps (0, 10, 20, …) so you can insert layers later without renumbering.
- `fixed: true` is the correct choice for HUD and UI layers — it is faster than pinning UI positions manually in `onUpdate`.
- `blendMode: 'additive'` on a particle or glow layer avoids dark halos on transparent sprites.
- Toggling `setVisible` is cheaper than destroying and re-adding entities when a layer is temporarily hidden.
