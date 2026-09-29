# Bitmap fonts: BitmapFontDef and FontRegistry

**Use this when** you want text drawn from a pre-rendered glyph atlas (an image plus a rectangle per glyph) instead of a system font, so it looks identical on every machine. This is what the GameMaker importer produces for a GameMaker font, and what `draw_text` uses after `draw_set_font`. A bitmap font is plain data (`BitmapFontDef`) registered in the game's `FontRegistry` (`game.fonts`).

---

## The data

```typescript
import type { BitmapFontDef } from '@emptysock/engine'

const menuFont: BitmapFontDef = {
  name: 'fnt_menu',
  atlasPath: './assets/fonts/fnt_menu.png',   // loaded like any other texture path
  size: 24,                                   // nominal size
  lineHeight: 37,                             // distance between baselines
  glyphs: {
    // keyed by Unicode code point
    65: { x: 2, y: 41, w: 18, h: 37, shift: 19, offset: 0 },   // 'A'
    66: { x: 22, y: 41, w: 16, h: 37, shift: 18, offset: 1 },  // 'B'
  },
  kerning: [[65, 66, -1]],   // [first, second, amount]: extra advance between that pair
}
```

- `x`, `y`, `w`, `h` locate the glyph rectangle in the atlas image, in pixels.
- `shift` is how far the pen advances after the glyph; `offset` is the left bearing applied when drawing the rectangle.
- `kerning` amounts are added to the advance between `first` and `second`.
- A character with no `glyphs` entry is skipped, not drawn as a fallback glyph.

## Registering and using it

```typescript
game.fonts.registerBitmap('fnt_menu', menuFont)

game.fonts.hasBitmap('fnt_menu')    // true
game.fonts.getBitmap('fnt_menu')    // the def
```

`registerBitmap` is independent of `register(id, descriptor)` (the CSS-style font descriptor used by `Label` widgets): an id can have either, or both. When both exist, GML `draw_text` prefers the bitmap; widgets keep using the descriptor.

Attach the registry to your renderer so it can find bitmap fonts: `RenderPipeline` takes `options.fonts`, or call `pipeline.attachFonts(game.fonts)`. With a GML runtime and a renderer, this is wired for you.

## Layout helpers (no renderer needed)

For measuring text or laying it out yourself, these are pure functions:

```typescript
import { layoutBitmapText, bitmapKerning } from '@emptysock/engine'

const layout = layoutBitmapText(menuFont, 'AB')
layout.width; layout.height
layout.placements   // [{ codePoint, glyph, x, y }, ...] relative to the text origin
bitmapKerning(menuFont, 65, 66)   // -1
```

## What to know

- **First draw may use a fallback.** Until the atlas image has loaded, text draws with the normal font for that frame and the load is started; the bitmap version appears once the image is ready. Preload the atlas (`AssetManifest`, `skills/27-asset-manifest.md`) to avoid the swap.
- **Colour tints the glyphs.** Atlases are drawn white-on-transparent and tinted by the current draw colour.
- **`Label` and the other UI widgets do not use bitmap fonts yet**; they draw from the CSS descriptor.
- **Alignment.** `fa_left` / `fa_center` / `fa_right` and the vertical equivalents work for bitmap and system fonts alike.

## Rules

| Wrong | Right |
|---|---|
| Registering a def whose `atlasPath` the game never serves | Make sure the atlas is a real asset at that path |
| Expecting missing characters to fall back to a default glyph | Add the glyph, or the character is not drawn |
| Editing `game.fonts` defs after text has drawn | Register once at startup; re-registering an id rebuilds its font |
