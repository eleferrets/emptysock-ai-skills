# RenderPipeline

**Use this when** you need something to actually show up on screen and don't want to hand-roll PixiJS wiring. `RenderPipeline` is the batteries-included renderer. It owns the raw renderer and a `LayerSystem` (draw order), and every `renderFrame(scene)` call walks the scene for entities carrying both `Transform` and `Sprite`, keeps each one's sprite in sync (position, rotation, scale, tint, alpha, anchor, visibility, layer, depth), loads its texture, and draws the frame. It also draws real tile sprites for a mounted `Tilemap`.

**Attaching `Transform` + `Sprite` to an entity is the entire contract for "this shows up on screen." There is no second, separate registration step.**

`RenderPipeline` only handles drawing at a fixed pixel size; pair it with `ViewportSystem` (see `skills/24-viewport-system.md`) to make that output fit the actual container across window sizes, orientations, and device pixel ratios: `RenderPipeline` draws, `ViewportSystem` scales what it drew.

---

## Setup

```typescript
import { Game, RenderPipeline } from '@emptysock/engine'

const render = new RenderPipeline()
await render.init({ width: 1280, height: 720 })   // options: width, height, backgroundColor, antialias, resolution, gpuTier, preference
document.body.appendChild(render.canvas)

game.attachRenderer(render)   // Game then calls render.renderFrame(main, overlays) each update
```

`RenderPipeline` implements `SceneRenderer`. Either attach it to the `Game` as above, or call `render.renderFrame(mainScene, overlayScenes?)` yourself once per frame after game logic. Use `syncEntities(scene)` instead in tests or a custom loop that renders separately. Free everything with `render.destroy()` on shutdown.

---

## Making something appear: Transform + Sprite

```typescript
import { Transform, Sprite } from '@emptysock/engine'

const player = scene.spawn('Player')
player.add(Transform, { x: 100, y: 200 })
player.add(Sprite, {
  texturePath: 'assets/hero.png',
  layer: 'foreground',   // named LayerSystem layer, defaults to 'default'
  depth: 10,              // draw order within the layer, defaults to 0
})

const sprite = player.get(Sprite)   // proxy | undefined
```

That is the whole setup. The next `renderFrame()` finds the entity, loads `hero.png`, places it on the `'foreground'` layer at depth `10`, and keeps its screen position in sync with `Transform`. Moving the entity is just moving its `Transform`:

```typescript
const transform = player.get(Transform)
if (transform !== undefined) transform.x += 50 * dt
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
| `frameCount` / `currentFrame` / `frameSpeed` / `loop` | `number` / `number` / `number` / `boolean` | `1` / `0` / `1` / `true` | Frame animation: with `frameCount > 1`, `texturePath` is a template containing `{n}` (for example `assets/hero/frame_{n}.png`), advanced by `SpriteAnimationSystem` |
| `width` / `height` | `number` | `0` | Pixel size; needed for nine-slice/tiled modes (`sliceMode` 1 / 2 with `sliceLeft/Right/Top/Bottom`) |
| `shader` | `string` | `''` | A `ShaderRegistry` shader id to render this sprite through |

Destroying the entity (`scene.destroy(entity)`) automatically removes and destroys its PixiJS sprite; there is nothing to clean up manually.

---

## Tilemaps

`RenderPipeline.mountTilemap()` draws a `Tilemap`'s tile grid as real textured sprites — `Tilemap` itself has no rendering of its own.

```typescript
import { TilemapSystem, AutoTileSystem } from '@emptysock/tilemap'

TilemapSystem.register(level1Data)                 // TilemapData
const tilemap = TilemapSystem.loadInto(scene, 'level1')

// Plain tiles:
render.mountTilemap(tilemap, 'background')

// Or resolve neighbour-aware tile variants through an AutoTileSystem
// (see skills/17-auto-tile.md; it satisfies the engine's AutoTileResolver interface):
const autoTile = new AutoTileSystem()
autoTile.addRuleSet(grassRuleSet)
render.mountTilemap(tilemap, 'background', autoTile)
```

Call `unmountTilemap(tilemap)` before mounting it again to rebuild after an edit (e.g. the level editor changed tiles at runtime), and on scene unload if you mounted it manually; `RenderPipeline.destroy()` unmounts everything it still holds.

---

## Layers and custom draw order

`RenderPipeline` creates its own `LayerSystem` unless you pass one in, and exposes it via the `layers` getter for defining project-specific layers:

```typescript
const render = new RenderPipeline()
render.layers.defineLayer('midground', 50)

// Or share an existing LayerSystem across RenderPipeline and other code:
import { LayerSystem } from '@emptysock/engine'
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
| Forgetting `renderFrame()`/`syncEntities()` (or `game.attachRenderer`) | Nothing shows up: `RenderPipeline` only syncs and draws when it is called |
| Skipping `render.destroy()` on shutdown | Always call it — it frees PixiJS sprites, mounted tilemaps, and the renderer itself |
| Expecting `Tilemap` to draw itself | It's pure data. `mountTilemap()` is what actually puts tiles on screen |
