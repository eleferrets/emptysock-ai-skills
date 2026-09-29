# Surfaces and blend modes (GML-style cutout lighting)

**Use this when** you are porting or writing a game that builds a *darkness overlay* the GameMaker way: fill an off-screen surface with a dark colour, cut holes in it for each light, then draw that surface over the view. The functions keep their GameMaker names (`surface_create`, `gpu_set_blendmode`, `draw_surface`, and so on) so imported GML calls them unchanged. They are exported from `@emptysock/engine`.

This is a separate path from `LightingSystem` (`skills/22-lighting-system.md`), not a replacement. Use `LightingSystem` (`LightSource` + `LightOccluder`) for engine-native lighting with shadows. Use this when you already have the GML surface technique or want its exact look.

---

## The pattern

```typescript
import {
  surface_create, surface_set_target, surface_reset_target, surface_free,
  draw_surface, draw_clear, gpu_set_blendmode, bm_normal, bm_subtract,
  draw_set_colour, draw_circle,
} from '@emptysock/engine'

// ctx is the GmlActionContext for your GML runtime (GmsProjectRuntime builds it for you);
// its `surfaces` field is what makes these calls do anything.
const dark = surface_create(ctx, 1280, 720)
const view = surface_create(ctx, 1280, 720)

// 1. Darkness surface: fill grey, subtract a light-shaped hole.
surface_set_target(ctx, dark)
const target = ctx.drawTarget            // now the surface's draw target (undefined without a backend)
draw_clear(ctx, 0x606060)
gpu_set_blendmode(ctx, bm_subtract)
if (target !== undefined) {
  draw_set_colour(target, 0xffffff)
  draw_circle(target, 640, 360, 120, false)
}
gpu_set_blendmode(ctx, bm_normal)
surface_reset_target(ctx)

// 2. Draw the darkness over the scene with subtract: only unlit areas get darker.
surface_set_target(ctx, view)
draw_clear(ctx, 0xffffff)
gpu_set_blendmode(ctx, bm_subtract)
draw_surface(ctx, dark, 0, 0)
gpu_set_blendmode(ctx, bm_normal)
surface_reset_target(ctx)
```

In real GML the `draw_*` calls take the current draw target implicitly; when transpiled they are threaded for you. Free surfaces you no longer need with `surface_free(ctx, id)`.

## API

| Function | Notes |
|---|---|
| `surface_create(ctx, w, h)` | Returns a surface id, or `-1` when there is no surface backend (headless). |
| `surface_exists(ctx, id)` / `surface_free(ctx, id)` | |
| `surface_set_target(ctx, id)` / `surface_reset_target(ctx)` | Redirects drawing into the surface until reset. Nests like GameMaker's target stack. Contents accumulate between targets unless you `draw_clear`. |
| `draw_surface(ctx, id, x, y)` | Draws the surface with the *current* blend mode. |
| `draw_clear(ctx, colour)` / `draw_clear_alpha(ctx, colour, alpha)` | Replaces the current target's contents with a solid colour. |
| `gpu_set_blendmode(ctx, mode)` / `draw_set_blend_mode(ctx, mode)` | `bm_normal`, `bm_add`, `bm_max`, `bm_subtract`. |
| `gpu_set_blendmode_ext(ctx, src, dest)` | Arbitrary blend factors are **not** supported: it warns once and does nothing. Use the four modes above. |

## What to know

- **`bm_subtract` needs a WebGL back buffer**, which the engine's renderer enables for you. If you build your own renderer, request one, otherwise subtract silently renders as a normal draw.
- **No backend, no effect.** Without `ctx.surfaces` (a headless run, or a context that was not given a renderer) every surface call is a safe no-op and `surface_create` returns `-1`.
- **`application_surface` is a sentinel**, not a real surface: `surface_get_width`/`surface_get_height` on it return the game's design resolution, and `surface_resize` on it changes that resolution.
- **Each subtract or add draw switches blend mode**, which breaks batching. A handful of lights is fine; hundreds of separately blended draws per frame need profiling on real hardware.

## Rules

| Wrong | Right |
|---|---|
| Forgetting `surface_reset_target` | Every `surface_set_target` needs its reset, or later drawing goes into the surface |
| Never calling `surface_free` on per-frame surfaces | Create once and reuse, or free when done: each surface is GPU memory |
| Using `gpu_set_blendmode_ext` and expecting factors to apply | Pick one of the four `bm_*` modes |
| Using this for a game with no GML lineage | Prefer `LightingSystem` |
