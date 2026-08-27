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

## Localisation (i18n)

```typescript
import { i18n } from '@emptysock/engine'

// Load locale bundles in onLoad:
await i18n.load('en', () => import('./locales/en.json'))
await i18n.load('fr', () => import('./locales/fr.json'))
await i18n.load('de', () => import('./locales/de.json'))

// Switch locale at runtime (e.g., from settings screen):
i18n.setLocale('fr')

// Translate a key:
i18n.t('menu.start')                // → "Commencer"
i18n.t('hud.score', { n: 1234 })   // → "Score : 1 234" (template substitution)
i18n.t('missing.key')              // → 'missing.key' (returns key, never throws)
```

### Locale JSON format

```json
{
  "menu.start": "Start Game",
  "menu.quit": "Quit",
  "hud.score": "Score: {{n}}",
  "hud.lives": "{{n}} lives remaining"
}
```

### LocalisationEditor panel

The IDE's LocalisationEditor panel manages these files visually:

- Add keys and values inline
- Add locale columns (en, fr, de, ja, etc.)
- Filter by key or value substring
- Import CSV (header: `key,en,fr,...`)
- Export CSV for spreadsheet editing or version control

Export from the panel, place the JSON files in `src/locales/`, then import them in `onLoad` as shown above.

### Notes
- `i18n.t()` falls back to returning the key itself when a translation is missing — it never throws. This means typos in key names are silent at runtime; use the LocalisationEditor's filter to catch missing translations before shipping.
- Locale bundles are loaded lazily. Only load the locales the user might select at startup — don't load all languages upfront.
- Template tokens use `{{name}}` syntax. Pass them as `{ name: value }` in the second argument.
