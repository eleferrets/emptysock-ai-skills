# Save System & Localisation

---

## SaveSystem

`SaveSystem` is generic key-value persistence for save slots, backed by `localStorage`. All operations are synchronous. It only knows how to store, retrieve, and validate an opaque JSON object per slot under a prefixed key — it does not decide what a "save slot" contains; that shape is supplied to the constructor as a Zod schema.

```typescript
import { SaveSystem, type GameSaveSlot } from '@emptysock/engine'

// With no schema, SaveSystem uses the default GameSaveSlot shape:
// { id, scene, data, timestamp, playtime }
const saves = new SaveSystem()

// save() only needs `scene` and `data` — timestamp defaults to Date.now(),
// playtime defaults to 0, and `id` is filled in from the slot name automatically:
saves.save('slot-1', {
  scene: 'Level3',
  data: { score: 8400, flags: { bossDefeated: true } },
})

// Load and validate — a non-null result is already guaranteed to match GameSaveSlot:
const slot = saves.load('slot-1')  // GameSaveSlot | null — never throws
if (slot !== null) {
  loadScene(slot.scene)
}

// List every stored slot that currently validates against the schema,
// sorted newest first:
const slots = saves.listSlots()  // GameSaveSlot[]

// Delete a slot — no-ops if it doesn't exist:
saves.delete('slot-1')
```

### `GameSaveSlot` — the default slot shape

```typescript
interface GameSaveSlot {
  readonly id: string
  readonly scene: string
  readonly data: Record<string, unknown>
  readonly timestamp: number
  readonly playtime: number
}
```

### Using a custom slot schema

A game whose save data doesn't fit `{ scene, data, timestamp, playtime }` passes its own Zod schema as the second constructor argument instead of relying on the default. The schema must validate the full slot including an `id: string` field — `save()` fills `id` in from the slot name automatically, so the schema just needs to require it.

```typescript
import { SaveSystem } from '@emptysock/engine'
import { z } from 'zod'

const CharacterSaveSchema = z.object({
  id: z.string(),
  characterName: z.string(),
  level: z.number().int().positive(),
  unlockedSkills: z.array(z.string()),
})
type CharacterSave = z.infer<typeof CharacterSaveSchema>

const characterSaves = new SaveSystem<CharacterSave>('char_save_', CharacterSaveSchema)

characterSaves.save('hero-1', {
  characterName: 'Aria',
  level: 5,
  unlockedSkills: ['dash', 'parry'],
})

const hero = characterSaves.load('hero-1')  // CharacterSave | null
```

The constructor's first argument, `prefix`, sets the `localStorage` key prefix (default `"emptysock_save_"`). Use a different prefix per `SaveSystem` instance to keep unrelated slot shapes from colliding in storage.

### Notes
- If the assembled slot does not validate against the configured schema, `save()` logs a warning and does not write anything — it never silently drops unknown fields or partially persists invalid data.
- `load()` returns `null` for a missing slot, corrupted stored JSON, or a slot that no longer matches the configured schema (e.g. written by an older game version with a different shape) — never throws.
- `listSlots()` skips malformed or foreign-shaped entries rather than throwing.
- Never cast a loaded slot `as MySaveType` — `load()` already validates against the schema you gave the constructor, so a non-null result is guaranteed to match `TSlot`.
- Slot names are arbitrary strings. Use a consistent naming convention (`slot-1`, `autosave`, `checkpoint-{level}`) to avoid collisions.
- `SaveSystem` has no `destroy()` method.
- To persist `VariableStore` state alongside a save, embed `variableStore.snapshot()` inside the slot's `data` field and call `variableStore.restore(...)` after loading — see `skills/13-variable-store.md`.

---

## Localisation

`LocalisationSystem` is instanced — create one in `onLoad` and keep a reference. Load locale data via `addTranslations()` before calling `setLocale()`.

```typescript
import { LocalisationSystem } from '@emptysock/engine'
import { z } from 'zod'

const TranslationMapSchema = z.record(z.string())

class GameScene extends Scene {
  private _localisation: LocalisationSystem | null = null

  override async onLoad(): Promise<void> {
    const localisation = new LocalisationSystem()

    // Load and validate each locale file before registering:
    const enRaw = await (await fetch('assets/i18n/en.json')).json()
    const frRaw = await (await fetch('assets/i18n/fr.json')).json()
    localisation.addTranslations('en', TranslationMapSchema.parse(enRaw))
    localisation.addTranslations('fr', TranslationMapSchema.parse(frRaw))

    localisation.setLocale('en')
    this._localisation = localisation
  }

  // Switch locale at runtime (e.g., from settings screen):
  setLanguage(code: string): void {
    this._localisation?.setLocale(code)
  }
}

// React to locale changes (e.g., refresh UI labels):
const unsub = localisation.onLocaleChange((locale) => {
  scoreLabel.text = localisation.t('hud.score', { score: this._score })
})
// In onDestroy:
unsub()

// Translate a key:
localisation.t('menu.start')                          // → "Start Game"
localisation.t('hud.score', { score: 1234 })         // → "Score: 1234" (template substitution)
localisation.t('missing.key')                         // → 'missing.key' (returns key, never throws)

// Read current locale:
const current = localisation.currentLocale            // string
```

### Locale JSON format

Place files at `assets/i18n/[locale].json` and fetch them in `onLoad`.

```json
{
  "menu.start": "Start Game",
  "menu.quit": "Quit",
  "hud.score": "Score: {{score}}",
  "hud.lives": "{{count}} lives remaining"
}
```

### LocalisationEditor panel

The IDE's LocalisationEditor panel manages these files visually:

- Add keys and values inline
- Add locale columns (en, fr, de, ja, etc.)
- Filter by key or value substring
- Import CSV (header: `key,en,fr,...`)
- Export CSV for spreadsheet editing or version control

Export from the panel and place the JSON files in `assets/i18n/`.

### Notes
- `localisation.t()` falls back to returning the key itself when a translation is missing — it never throws. Typos in key names are silent at runtime; use the LocalisationEditor's filter to catch missing translations before shipping.
- Template tokens use `{{name}}` syntax. Pass them as `{ name: value }` in the second argument.
- `setLocale()` takes an IETF language tag string (`'en'`, `'fr'`, `'ja'`). The locale must correspond to a locale you have registered with `addTranslations()`.
- Always validate fetched JSON with a Zod schema before passing it to `addTranslations()` — locale files can be corrupt or edited externally.
