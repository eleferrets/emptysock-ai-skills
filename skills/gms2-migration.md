# GMS2 Migration

**Use this when** you're bringing a GameMaker Studio 2 project over to EmptySock. Work through the steps in order. The importer's been validated end to end against a real, moderately complex GMS2 export (59 objects, 202 sprites, 10 rooms) — it actually converts objects, scripts, sprites, and rooms now, not just objects and scripts. That said, GML → TypeScript logic translation always needs a manual review pass, even for the events it transpiles automatically.

---

## Step 1: Run the importer

Point the toolchain at the `.yyp` project file:

```bash
emptysock-toolchain import --from gms2 --project path/to/game.yyp --out ./my-game/
```

This generates TypeScript stubs under `./my-game/src/` and writes `migration-report.md` in the output directory. Add `--dry-run` to preview output without writing files.

---

## Step 2: Read migration-report.md

Open `migration-report.md` before touching any stub. It lists:

- Every unresolved GML built-in, by file and line number.
- Suggested EmptySock equivalents where a mapping exists.
- Room dimensions and layer structure for manual reconstruction (tile layers and instance placement are read, but only instance placement is re-emitted).
- Extracted shader files that need porting to `PostProcessSystem`.

Fix the highest-confidence mappings first and leave the uncertain ones for later — no point guessing your way through a mapping the report itself isn't sure about.

Object logic is regex-transpiled where a direct pattern match exists, and this now covers more than `Create`/`Step`/`Draw`/`Destroy`: collision events (`Collision_<other object>.gml`) become `onCollideWith<Other>()`, and keyboard events (`KeyPress_<vk code>.gml` / `KeyRelease_<vk code>.gml`) become `onKeyPress<Name>()` / `onKeyRelease<Name>()`. Object inheritance (a GMS2 object's parent) carries over as TypeScript class extension in the generated stubs.

Projects migrated up from GameMaker 8.1 (or still using its DnD-compatibility layer) call GM8-era internals — `action_move`, `action_sprite_set`, `gml_pragma`, `__global_object_depths` — that are left as unresolved identifiers for manual review rather than guessed at; they are distinct from ordinary modern-GML gaps.

---

## Step 3: Map GML patterns to TypeScript

Common patterns:

| GML | TypeScript / EmptySock |
|-----|------------------------|
| `instance_create_layer(x, y, layer, obj)` | `scene.createEntity()` + `addComponent` |
| `instance_destroy()` | `entity.destroy()` |
| `alarm[0] = 60` | `entity.startCoroutine(waitFrames(60, fn))` |
| `hspeed` / `vspeed` | `PhysicsBody` velocity |
| `sprite_index` / `image_index` | `Sprite` + `Animator` components |
| `global.variable` | Module-level `let` / `const` |
| `with (obj_enemy) { ... }` | `scene.query(EnemyComponent).forEach(...)` |
| `room_goto(rm_next)` | `SceneManager.load('SceneName')` |
| `audio_play_sound(snd, priority, loop)` | `AudioSystem.play('name', { loop })` |
| `draw_sprite(spr, img, x, y)` | `Sprite` component (declarative) |
| `irandom(n)` | `Math.floor(Math.random() * (n + 1))` |

For the full table see `docs/manual/11-gms2-migration.md`.

---

## Step 4: Use the GML compat shim during migration

Import the shim to avoid rewriting utility calls immediately:

```typescript
import * as GML from '@emptysock/engine/compat';

GML.lerp(a, b, t);
GML.clamp(v, lo, hi);
GML.irandom(n);
GML.ds_map_create();        // returns a Map<string, unknown>
GML.ds_map_set(m, k, v);
GML.ds_map_find_value(m, k);
GML.ds_list_create();       // returns an Array<unknown>
GML.ds_list_add(l, v);
GML.ds_list_find_value(l, i);
```

Replace shim calls with idiomatic TypeScript as each object settles down. The shim is a bridge to get you across, not something to ship.

---

## Asset status quick reference

| GMS2 asset | Status |
|------------|--------|
| Objects → Components | Auto-stub; manual GML migration |
| Scripts → TS modules | Auto-stub; manual GML migration |
| Rooms → Scenes | Auto-converted — each room becomes a `loadRoom<Name>()` function that spawns one entity per instance placement, wired to the imported object classes. Tile layers and per-instance creation code are not reconstructed. |
| Sprites | Auto-converted — each frame's PNG is copied to `assets/sprites/<name>/` with a generated `.sprite.ts` descriptor for the `Sprite` component |
| Sounds | Manual — listed in the migration report; use `AudioSystem.play` |
| Tilesets / Tilemaps | Manual — room tile grids are read but not re-emitted; recreate in the TilemapEditor panel |
| Sequences | Manual — use Sequence Editor panel |
| Paths | Manual — use `PathfindingSystem` |
| Shaders | Manual — port to `PostProcessSystem` custom effect |

---

## Common pitfalls

- **Dynamic typing:** GML stubs compile, but they'll be riddled with implicit `any` chains. Go add real TypeScript types, file by file.
- **Room system:** GMS2's room/depth/layer model doesn't map 1:1 onto EmptySock's `LayerSystem`. Don't expect it to.
- **`ds_grid`, `ds_priority`, `ds_stack`:** Nothing equivalent exists — reach for TypeScript arrays and `Map` instead.
- **GML event order (Step Begin / Step / Step End):** If the order matters, split the logic into separate coroutines or actor messages.
- **`persistent` objects:** A module-level singleton, or `SceneManager.load()` options, carries state across scenes the same way.
- **`async` events (HTTP, dialog):** Use `fetch` directly, and re-enter game logic through actor messages.
