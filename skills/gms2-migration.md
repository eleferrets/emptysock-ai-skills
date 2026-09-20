# GMS2 Migration

This skill guides migrating a GameMaker Studio 2 project to EmptySock Engine. Work through the steps in order. The importer has been validated end to end against a real, moderately complex GMS2 export (59 objects, 202 sprites, 10 rooms): it now actually converts objects, scripts, sprites, and rooms — not just objects and scripts. GML → TypeScript logic translation is still always manual review, even for the events it transpiles automatically.

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

Fix the highest-confidence mappings first; leave uncertain ones for later.

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

Replace shim calls with idiomatic TypeScript as each object stabilises. The shim is a migration bridge, not a production dependency.

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

- **Dynamic typing:** GML stubs compile but will have implicit `any` chains. Add TypeScript types file by file.
- **Room system:** GMS2's room/depth/layer model differs from EmptySock's `LayerSystem`. Do not expect a 1:1 mapping.
- **`ds_grid`, `ds_priority`, `ds_stack`:** No equivalents — use TypeScript arrays and `Map`.
- **GML event order (Step Begin / Step / Step End):** Split into separate coroutines or actor messages if order matters.
- **`persistent` objects:** Use a module-level singleton or `SceneManager.load()` options to carry state across scenes.
- **`async` events (HTTP, dialog):** Use `fetch` directly; re-enter game logic via actor messages.
