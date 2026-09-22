# RenderPipeline

**Use this when** you need something to actually show up on screen and don't want to hand-roll PixiJS wiring. `RenderPipeline` is the batteries-included renderer. It owns a `RenderSystem` (the raw renderer) and a `LayerSystem` (draw order), and every `renderFrame(scene)` call walks the scene for entities carrying both `Transform` and `Sprite`, keeps each one's sprite in sync (position, rotation, scale, tint, alpha, anchor, visibility, layer, depth), loads its texture, and draws the frame. It also draws real tile sprites for a mounted `Tilemap`.

**Attaching `Transform` + `Sprite` to an entity is the entire contract for "this shows up on screen." There is no second, separate registration step.**

`RenderPipeline` only handles drawing at a fixed pixel size — pair it with `ViewportSystem` (see `skills/24-viewport-system.md`) to make that output fit the actual container across window sizes, orientations, and device pixel ratios: `RenderPipeline` draws, `ViewportSystem` scales what it drew.

---

## Setup (in onLoad)

```typescript
import { RenderPipeline } from '@emptysock/engine'

class GameScene extends Scene {
  private _render = new RenderPipeline()

  override async onLoad(): Promise<void> {
    await this._render.init({ width: 1280, height: 720 })
    document.body.appendChild(this._render.canvas)
  }

  override onUpdate(): void {
    // Nothing to do here — call renderFrame once per frame after update logic.
  }

  override onDestroy(): void {
    this._render.destroy()
  }
}
```

`RenderPipeline` does not run itself — call `renderFrame(scene)` once per frame, after your game logic and `SceneManager` update, typically from the engine's render step or the tail of `onUpdate`:

```typescript
override onUpdate(dt: number): void {
  // ...game logic...
  this._render.renderFrame(this)   // syncs Transform+Sprite entities, then renders
}
```

Use `syncEntities(scene)` instead of `renderFrame(scene)` in tests or a custom loop that renders separately.

---

## Making something appear — Transform + Sprite

```typescript
import { Transform, Sprite } from '@emptysock/engine'

const player = scene.createEntity('Player')
player.addComponent(new Transform({ x: 100, y: 200 }))
player.addComponent(new Sprite({
  texturePath: 'assets/hero.png',
  layer: 'foreground',   // named LayerSystem layer — defaults to 'default'
  depth: 10,              // draw order within the layer — defaults to 0
}))

// Read it back with the typed component-type token:
const sprite = player.getComponent(Sprite.TYPE)   // Sprite | undefined
```

That is the whole setup. The next call to `renderFrame()` finds the entity, loads `hero.png`, places it on the `'foreground'` layer at depth `10`, and keeps its screen position in sync with `Transform` every frame after. Moving the entity is just moving its `Transform`:

```typescript
const transform = player.requireComponent(Transform.TYPE)
transform.x += 50 * dt
```

### `Sprite` fields

| Field | Type | Default | Notes |
|---|---|---|---|
| `texturePath` | `string` | `''` | Empty string renders a white square placeholder instead of loading anything |
| `tint` | `number` | `0xffffff` | Hex colour multiplier |
| `alpha` | `number` | `1` | |
| `anchorX` / `anchorY` | `number` | `0.5` | 0–1, fraction of the sprite's own size |
| `layer` | `string` | `'default'` | Any `LayerSystem` layer name — see `skills/layer-system.md` |
| `depth` | `number` | `0` | Draw order within `layer` — lower draws first (behind) |
| `visible` | `boolean` | `true` | Toggle without removing the component |

Destroying the entity (`entity.destroy()`) automatically removes and destroys its PixiJS sprite — there is nothing to clean up manually.

---

## Tilemaps

`RenderPipeline.mountTilemap()` draws a `Tilemap`'s tile grid as real textured sprites — `Tilemap` itself has no rendering of its own.

```typescript
import { TilemapSystem, AutoTileSystem } from '@emptysock/engine'

const tilemap = TilemapSystem.loadInto(this, 'level1')

// Plain tiles:
this._render.mountTilemap(tilemap, 'background')

// Or resolve neighbour-aware tile variants through an AutoTileSystem
// (see skills/17-auto-tile.md):
const autoTile = new AutoTileSystem()
autoTile.addRuleSet(grassRuleSet)
this._render.mountTilemap(tilemap, 'background', autoTile)
```

Call `unmountTilemap(tilemap)` before mounting it again to rebuild after an edit (e.g. the level editor changed tiles at runtime), and in `onDestroy` if you mounted it manually — `RenderPipeline.destroy()` unmounts everything it still holds.

---

## Layers and custom draw order

`RenderPipeline` creates its own `LayerSystem` unless you pass one in, and exposes it via the `layers` getter for defining project-specific layers:

```typescript
const render = new RenderPipeline()
render.layers.defineLayer('midground', 50)

// Or share an existing LayerSystem across RenderPipeline and other code:
const layers = new LayerSystem()
const render2 = new RenderPipeline({ layers })
```

See `skills/layer-system.md` for the full `LayerSystem` API — most games only need `defineLayer` and `setVisible` directly; per-entity placement is handled automatically from `Sprite.layer` / `Sprite.depth`.

---

## Custom texture loading

Pass `textureLoader` to resolve texture paths yourself instead of the default (`Assets.load`) — useful for a virtual filesystem, a CDN prefix, or tests:

```typescript
const render = new RenderPipeline({
  textureLoader: async (path) => myAssetPipeline.resolve(path),
})
```

---

## Rules

| Wrong | Right |
|---|---|
| Manually creating a PixiJS `Sprite` and adding it to a container | Attach `Transform` + `Sprite` components and let `RenderPipeline` draw it |
| Calling `layers.addEntity()` for a rendered entity | `RenderPipeline` already does this for you, every frame, from `Sprite.layer`/`Sprite.depth` |
| Forgetting `renderFrame()`/`syncEntities()` each frame | Nothing shows up — `RenderPipeline` only syncs and draws when you actually call it |
| Skipping `render.destroy()` in `onDestroy` | Always call it — it frees PixiJS sprites, mounted tilemaps, and the renderer itself |
| `entity.getComponent(Sprite)` | `entity.getComponent(Sprite.TYPE)` — infers `Sprite \| undefined` and catches typos for you |
| Expecting `Tilemap` to draw itself | It's pure data. `mountTilemap()` is what actually puts tiles on screen |
