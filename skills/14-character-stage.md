# CharacterStage

`CharacterStage` manages left/centre/right position slots for visual novel character sprites. Characters fade in and out, can be swapped per expression variant, and are rendered via Canvas 2D before the UISystem pass.

---

## Quick start

```typescript
import { CharacterStage, VNBackgroundLayer, VNSystem, VNTextbox, UISystem } from '@emptysock/engine'

class NarrativeScene extends Scene {
  private _vn = new VNSystem()
  private _bg!: VNBackgroundLayer
  private _stage!: CharacterStage
  private _textbox!: VNTextbox

  async onLoad() {
    await this._vn.load({ startNode: 'intro', nodes: { /* ... */ } })

    this._bg = new VNBackgroundLayer({ canvasWidth: 800, canvasHeight: 600 })
    this._bg.setBackground('assets/rooms/forest.jpg')

    this._stage = new CharacterStage({ canvasWidth: 800, canvasHeight: 600 })
    this._stage.show('left', 'assets/sprites/hero_neutral.png')
    this._stage.show('right', 'assets/sprites/guard_neutral.png', { fadeDuration: 0.4 })

    this._textbox = new VNTextbox({ canvasWidth: 800, canvasHeight: 600 })
    this._textbox.bind(this._vn)
  }

  onUpdate(dt: number) {
    this._bg.update(dt)
    this._stage.update(dt)
    UISystem.update(dt)

    // Render order: background → characters → UI
    const ctx = /* your CanvasRenderingContext2D */
    this._bg.render(ctx)
    this._stage.render(ctx)
    UISystem.render(ctx, 800, 600)
  }

  onDestroy() {
    this._textbox.destroy()
    this._stage.clear()
  }
}
```

---

## CharacterStage API

```typescript
new CharacterStage(opts: CharacterStageOptions)

stage.show(slot: StageSlot, imagePath: string, opts?: CharacterShowOptions): void
stage.hide(slot: StageSlot, fadeDuration?: number): void
stage.update(dt: number): void
stage.render(ctx: CanvasRenderingContext2D): void
stage.clear(): void
```

### CharacterStageOptions

| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `canvasWidth` | `number` | required | Canvas pixel width |
| `canvasHeight` | `number` | required | Canvas pixel height |
| `baselineY` | `number` | `0.85` | Vertical baseline as fraction of canvas height |
| `maxHeightFraction` | `number` | `0.7` | Max character height as fraction of canvas height |

### StageSlot

`'left' | 'center' | 'right'`

### CharacterShowOptions

| Option | Type | Default |
|--------|------|---------|
| `expression` | `string` | `''` |
| `fadeDuration` | `number` | `0.3` |

---

## VNBackgroundLayer API

```typescript
new VNBackgroundLayer(opts: VNBackgroundLayerOptions)

bg.setBackground(imagePath: string, opts?: { fadeDuration?: number; fit?: 'cover' | 'contain' | 'stretch' }): void
bg.clearBackground(fadeDuration?: number): void
bg.showCG(imagePath: string, opts?: { fadeDuration?: number; fit?: 'cover' | 'contain' | 'stretch' }): void
bg.hideCG(fadeDuration?: number): void
bg.update(dt: number): void
bg.render(ctx: CanvasRenderingContext2D): void
```

### VNBackgroundLayerOptions

| Option | Type | Default |
|--------|------|---------|
| `canvasWidth` | `number` | required |
| `canvasHeight` | `number` | required |
| `fadeDuration` | `number` | `0.5` |

---

## Render order

Always render in this order per frame:

1. `VNBackgroundLayer.render(ctx)` — background fill and CG overlay
2. `CharacterStage.render(ctx)` — character sprites
3. `UISystem.render(ctx, w, h)` — textbox and HUD

---

## Rules

- Call `CharacterStage.update(dt)` and `VNBackgroundLayer.update(dt)` each frame to tick fade animations.
- `CharacterStage` and `VNBackgroundLayer` draw directly to a `CanvasRenderingContext2D` — they do not use the PixiJS renderer.
- Do not call `show()` inside `onUpdate()` — call it in response to VNSystem node transitions.
- `clear()` removes all characters immediately (no fade). Call it in `onDestroy()`.
