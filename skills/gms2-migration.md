# GMS2 Migration

This skill guides migrating a GameMaker Studio 2 project to EmptySock Engine. Work through the steps in order; the importer handles asset discovery, but GML → TypeScript translation is always manual.

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
- Room dimensions and layer structure for manual reconstruction.
- Extracted shader files that need porting to `PostProcessSystem`.

Fix the highest-confidence mappings first; leave uncertain ones for later.

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
| Rooms → Scenes | Manual — recreate in EmptySock |
| Sprites | Manual — copy files, import via Asset Browser |
| Sounds | Manual — use `AudioSystem.play` |
| Tilesets / Tilemaps | Manual — use TilemapEditor panel |
| Sequences | Manual — use Sequence Editor panel |
| Paths | Manual — use `NavMeshSystem` |
| Shaders | Manual — port to `PostProcessSystem` custom effect |

---

## Common pitfalls

- **Dynamic typing:** GML stubs compile but will have implicit `any` chains. Add TypeScript types file by file.
- **Room system:** GMS2's room/depth/layer model differs from EmptySock's `LayerSystem`. Do not expect a 1:1 mapping.
- **`ds_grid`, `ds_priority`, `ds_stack`:** No equivalents — use TypeScript arrays and `Map`.
- **GML event order (Step Begin / Step / Step End):** Split into separate coroutines or actor messages if order matters.
- **`persistent` objects:** Use a module-level singleton or `SceneManager.load()` options to carry state across scenes.
- **`async` events (HTTP, dialog):** Use `fetch` directly; re-enter game logic via actor messages.
