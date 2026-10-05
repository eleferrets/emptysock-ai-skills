# Bitmap fonts: BitmapFontDef and FontRegistry

**Use this when** you want text drawn from a pre-rendered glyph atlas (an image plus a rectangle per glyph) instead of a system font, so it looks identical on every machine. A bitmap font is plain data (`BitmapFontDef`) registered in the game's `FontRegistry` (`game.fonts`).

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

`registerBitmap` is independent of `register(id, descriptor)` (the CSS-style font descriptor used by `Label` widgets): an id can have either, or both.

Attach the registry to your renderer so it can find bitmap fonts: `RenderPipeline` takes `options.fonts`, or call `pipeline.attachFonts(game.fonts)`.

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

- **First draw may use a fallback.** Until the atlas image has loaded, text draws with the normal font for that frame and the load is started; the bitmap version appears once the image is ready. Load the atlas early (for example `TextureStore.load(atlasPath)`, see `skills/42-texture-store.md`) to avoid the swap.
- **Atlas format.** Atlases are expected to be white-on-transparent glyph sheets; `RenderPipeline` text uses them through pixi bitmap fonts, while `UISystem` only draws them untinted.
- **UI widgets can use bitmap fonts.** Set `fontId` on a `Label` (or button/checkbox) to the registered id and give `UISystem` the registry (`new UISystem(tree, { fonts: game.fonts })`). Bitmap drawing only happens for the default white text colour; coloured text falls back to the CSS font. Without a bitmap def, `fontId` resolves through the CSS descriptor registered with `fonts.register(id, descriptor)`.
- **Alignment.** `Label.align` (0 left, 1 centre, 2 right) works for bitmap and system fonts alike.

## Rules

| Wrong | Right |
|---|---|
| Registering a def whose `atlasPath` the game never serves | Make sure the atlas is a real asset at that path |
| Expecting missing characters to fall back to a default glyph | Add the glyph, or the character is not drawn |
| Editing `game.fonts` defs after text has drawn | Register once at startup; re-registering an id rebuilds its font |
