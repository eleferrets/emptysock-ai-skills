# LayerSystem

`LayerSystem` controls draw order: entities are assigned to a named layer at an explicit depth; `RenderSystem` draws layers in ascending index order, then entities within a layer in ascending depth. Four built-in layers are created by the constructor — add more with `defineLayer`.

## Import

```typescript
import { LayerSystem, LAYER, type LayerConfig, type LayerSortKey } from '@emptysock/engine'
```

## Built-in layer constants

```typescript
LAYER.BACKGROUND  // -1000 — furthest back
LAYER.DEFAULT     //     0 — default when no layer assigned
LAYER.FOREGROUND  //   100
LAYER.UI          //  1000 — drawn on top
```

The constructor pre-registers layers named `'background'`, `'default'`, `'foreground'`, and `'ui'` at these indices.

## Setup (in onLoad)

```typescript
private _layers!: LayerSystem

override onLoad(): void {
  this._layers = new LayerSystem()

  // Optional: add project-specific layers
  this._layers.defineLayer('midground', 50)
  this._layers.defineLayer('fx', 80)
}

override onDestroy(): void {
  this._layers.destroy()
}
```

## Assigning entities to layers

```typescript
// Assign entity to a built-in layer at default depth (0):
this._layers.addEntity(player.id, 'foreground')

// Assign with explicit depth — lower depth draws first (behind):
this._layers.addEntity(treeBack.id,  'midground', -10)
this._layers.addEntity(treeFront.id, 'midground',  10)

// Move an entity's depth without changing layer:
this._layers.setDepth(treeBack.id, -20)

// Unregister entity (entity still exists — just excluded from sort):
this._layers.removeEntity(oldEntity.id)
```

## Layer visibility

```typescript
this._layers.setVisible('fx', false)    // cull whole layer from render
this._layers.setVisible('fx', true)

const visible = this._layers.isVisible('fx')  // boolean
```

## Sorting and introspection

`RenderSystem` uses `getSortKey` to sort draw calls each frame:

```typescript
const [layerIndex, depth] = this._layers.getSortKey(entity.id)

// Sort an entity array for rendering:
entities.sort((a, b) => {
  const [al, ad] = this._layers.getSortKey(a.id)
  const [bl, bd] = this._layers.getSortKey(b.id)
  return al !== bl ? al - bl : ad - bd
})

// All entities on one layer, sorted by depth:
const onMidground = this._layers.getEntitiesOnLayer('midground')
// → Array<{ entityId: number; depth: number }>

// All layer configs sorted by index (render order):
const sorted = this._layers.getLayersSorted()
// → LayerConfig[] — each has { name, index, visible }
```

## API reference

| Method / Property | Signature | Notes |
|---|---|---|
| `defineLayer` | `(name: string, index: number): void` | Define or redefine a layer. Lower index = drawn behind. |
| `addEntity` | `(entityId: number, layerName: string, depth?: number): void` | Assign entity to a layer. Unknown layer falls back to `'default'`. |
| `removeEntity` | `(entityId: number): void` | Unregister entity from the layer system. |
| `setDepth` | `(entityId: number, depth: number): void` | Update depth within the entity's current layer. |
| `getEntityLayer` | `(entityId: number): string \| null` | Current layer name, or null if unregistered. |
| `getEntityDepth` | `(entityId: number): number` | Current depth, or 0 if unregistered. |
| `getSortKey` | `(entityId: number): LayerSortKey` | `[layerIndex, depth]` — pass to a sort comparator. |
| `getLayerIndex` | `(name: string): number` | Index of a named layer, or `LAYER.DEFAULT` if unknown. |
| `setVisible` | `(name: string, visible: boolean): void` | Show/hide an entire layer. |
| `isVisible` | `(name: string): boolean` | Returns true if the layer is visible. |
| `getEntitiesOnLayer` | `(layerName: string): Array<{entityId, depth}>` | Entities on a layer sorted by depth ascending. |
| `getLayersSorted` | `(): LayerConfig[]` | All layer configs sorted by index ascending. |
| `destroy` | `(): void` | Clear all placements and layers. Call in onDestroy. |

## Notes

- Use index gaps (−1000, 0, 50, 80, 100, 1000) so you can insert layers later without renumbering everything.
- `depth` within a layer controls fine-grained draw order (which tree is in front of which). Use it instead of changing a layer.
- Unregistered entities (never passed to `addEntity`) get `LAYER.DEFAULT, depth 0` from `getSortKey`.
