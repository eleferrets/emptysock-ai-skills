# Save System & Localisation

---

## SaveSystem

Persists typed game data to disk (Tauri) or localStorage (browser). Always validate loaded data with a schema — files can be corrupt or from a different game version.

```typescript
import { SaveSystem } from '@emptysock/engine'
import { z } from 'zod'

// Define and validate your save shape:
const Schema = z.object({
  scene:  z.string(),
  score:  z.number(),
  flags:  z.record(z.boolean()),
  level:  z.number().default(1),
})
type SaveData = z.infer<typeof Schema>

// Save:
await SaveSystem.save('slot-1', {
  scene: 'Level3',
  score: 8400,
  flags: { bossDefeated: true },
  level: 3,
})

// Load and validate:
const raw  = await SaveSystem.load('slot-1')   // throws if slot not found
const data = Schema.parse(raw.data)             // throws on corrupt / schema mismatch

// List available slots:
const slots = await SaveSystem.listSlots()      // string[]

// Delete a slot:
await SaveSystem.delete('slot-1')
```

### Notes
- `SaveSystem.load()` throws `SlotNotFoundError` if the slot does not exist. Check `listSlots()` first or wrap in try/catch.
- Never cast `raw.data as MySaveType` — schema validation is the contract.
- Slot names are arbitrary strings. Use a consistent naming convention (`slot-1`, `autosave`, `checkpoint-{level}`) to avoid collisions.
- In browser mode, data is stored in `localStorage`. Clearing site data deletes saves.

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
