# CharacterStage

**Use this when** you're staging character sprites for a visual novel scene — left/centre/right slots, fade transitions, expression swaps. `CharacterStage` handles all of it, rendering via Canvas 2D ahead of the UISystem pass.

---

## Quick start

```typescript
import { UISystem, CharacterStage } from '@emptysock/engine'
import {
  VNBackgroundLayer,
  VNSystem,
  VNTextbox,
  storyGraphToDialogueTree,
  type StoryGraph,
} from '@emptysock/vn'

class NarrativeScene extends Scene {
  private _vn: VNSystem | null = null
  private _bg: VNBackgroundLayer | null = null
  private _stage: CharacterStage | null = null
  private _textbox: VNTextbox | null = null

  override async onLoad(): Promise<void> {
    const response = await fetch('assets/story/chapter1.storyGraph.json')
    const graph: StoryGraph = await response.json() as StoryGraph
    const tree = storyGraphToDialogueTree(graph)

    this._vn = new VNSystem()
    this._vn.load(tree)

    this._bg = new VNBackgroundLayer({ canvasWidth: 800, canvasHeight: 600 })
    this._bg.setBackground('assets/rooms/forest.jpg')

    this._stage = new CharacterStage({ canvasWidth: 800, canvasHeight: 600 })
    this._stage.show('left', 'assets/sprites/hero_neutral.png')
    this._stage.show('right', 'assets/sprites/guard_neutral.png', { fadeDuration: 0.4 })

    this._textbox = new VNTextbox({ canvasWidth: 800, canvasHeight: 600 })
    this._textbox.bind(this._vn)
  }

  onUpdate(dt: number) {
    if (this._bg === null || this._stage === null) return
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
    this._textbox?.destroy()
    this._stage?.clear()
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
stage.render(ctx: CanvasRenderingContext2D): void  // ctx is a browser CanvasRenderingContext2D
stage.clear(): void  // immediately removes all characters from all slots
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
bg.clearCG(fadeDuration?: number): void
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

- Call `CharacterStage.update(dt)` and `VNBackgroundLayer.update(dt)` every frame, or fade animations just sit frozen.
- `CharacterStage` and `VNBackgroundLayer` draw straight to a `CanvasRenderingContext2D` — no PixiJS renderer involved.
- Don't call `show()` inside `onUpdate()`. Call it in response to VNSystem node transitions, where it belongs.
- `clear()` removes every character instantly, no fade. Call it in `onDestroy()`.
