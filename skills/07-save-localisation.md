# Save System & Localisation

---

## SaveSystem

Saving is covered in full by `skills/33-save-system.md` (component-based `SaveSystem(scene, components, options)`, async, adapter-backed). There is no separate standalone slot-schema `SaveSystem` and no `GameSaveSlot` type. To persist `VariableStore` and `GlobalStore` state, pass them as `variables` / `globals` options (see skill 33), or call `variables.snapshot()` / `variables.restore(...)` yourself (see `skills/13-variable-store.md`).

---

## Localisation

**Use this when** your game needs to speak more than one language. `LocalisationSystem` is instanced: each scene's lifecycle context already provides one as `ctx.localisation`, so you rarely construct your own. Load locale data with `addTranslations()` before calling `setLocale()`.

```typescript
import { defineScene } from '@emptysock/engine'

export const MenuScene = defineScene({
  async onLoad(scene, { localisation }) {
    const en: Record<string, string> = await (await fetch('assets/i18n/en.json')).json()
    const fr: Record<string, string> = await (await fetch('assets/i18n/fr.json')).json()
    localisation.addTranslations('en', en)
    localisation.addTranslations('fr', fr)
    localisation.setLocale('en')

    const unsub = localisation.onLocaleChange((locale) => {
      // refresh UI labels
    })
    // call unsub() in onUnload
  },
})
```

Standalone use works too: `import { LocalisationSystem } from '@emptysock/engine'; const loc = new LocalisationSystem()`.

### API

| Member | Signature |
|--------|-----------|
| `setLocale` | `(locale: string): void` (fires change handlers) |
| `currentLocale` | `string` getter (default `'en'`) |
| `addTranslations` | `(locale: string, map: Record<string, string>): void` (merges over existing keys) |
| `onLocaleChange` | `(handler: (locale: string) => void): () => void` (returns unsubscribe) |
| `t` | `(key: string, vars?: Record<string, string \| number>): string` |
| `destroy` | `(): void` |

```typescript
localisation.t('menu.start')                  // "Start Game"
localisation.t('hud.score', { score: 1234 })  // "Score: 1234"
localisation.t('missing.key')                 // 'missing.key' (returns the key, never throws)
```

### Locale JSON format

```json
{
  "menu.start": "Start Game",
  "hud.score": "Score: {{score}}",
  "hud.lives": "{{count}} lives remaining"
}
```

### Notes
- `t()` returns the key itself when a translation is missing, so a typo'd key fails silently. Check coverage before shipping.
- Template tokens use `{{name}}`; pass values as `{ name: value }`.
- `setLocale()` does not validate that the locale was registered; an unregistered locale makes every `t()` return its key.
- Validate fetched JSON (for example with Zod) before `addTranslations()`; locale files get hand-edited.
