# GlobalStore and GameGlobals

**Use this when** you need a named, game-wide value that any scene can read or write: a score, the current difficulty, a flag set in one scene and read in the next. `GlobalStore` is a `Game`-owned map of arbitrary names to arbitrary values (`game.globals`, `ctx.globals` inside a scene). One instance per `Game`, alive for its whole lifetime, and it survives scene changes.

It is not `VariableStore`. `VariableStore` is the numbered, integer-only, 1-1000 variables-and-switches store used by story and map-event conditions (`skills/13-variable-store.md`). Use `GlobalStore` for anything with a real name or a non-integer value.

---

## Basic use

```typescript
export const Hud = defineScene({
  onLoad(_scene, ctx) {
    ctx.globals.set('score', 0)
    ctx.globals.set('difficulty', 'hard')
  },
  onUpdate() { /* ... */ },
})

// Later, in any scene:
const score = ctx.globals.get<number>('score') ?? 0
ctx.globals.set('score', score + 10)
ctx.globals.has('score')      // true
ctx.globals.delete('score')
[...ctx.globals.keys()]       // every name currently set
ctx.globals.clear()           // reset (mainly for tests)
```

`get` returns `undefined` for a name that was never set, so handle that case.

---

## Typing names with `GameGlobals`

`GameGlobals` is an empty interface you augment with declaration merging. Declared names then type-check and autocomplete on `get`/`set`; anything undeclared still works through the untyped overloads.

```typescript
declare module '@emptysock/engine' {
  interface GameGlobals {
    score: number
    difficulty: 'easy' | 'normal' | 'hard'
  }
}

ctx.globals.set('score', 5)          // ok
ctx.globals.set('score', 'five')     // type error
const d = ctx.globals.get('difficulty')   // 'easy' | 'normal' | 'hard' | undefined
```

In the IDE, the **Game Globals** panel edits these declarations (a name plus a TypeScript type expression each). It writes the augmentation into the Code editor's types for you and saves the declarations in the project file. Only the *declarations* are saved; runtime values are not persisted (use `SaveSystem`, `skills/33-save-system.md`, for that).

## Rules

| Wrong | Right |
|---|---|
| `export const settings = { ... }` as a module-level global | `ctx.globals.set('settings', { ... })`: one instance per `Game`, isolated between tests |
| Putting a float or a string into `VariableStore` | `GlobalStore` holds any value; `VariableStore` truncates to integers |
| Assuming a global survives a restart | It doesn't; persist through `SaveSystem` |
| Reading `ctx.globals.get('x')` and using it without a check | It is `undefined` until set; default it: `?? 0` |
