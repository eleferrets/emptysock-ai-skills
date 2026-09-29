# TextureStore

**Use this when** you are writing a host, tool or system that needs to load a texture by path and look it up again synchronously, and you want it to share the cache the engine's own rendering and UI already use. `TextureStore` is that one shared load-and-lookup path for sprites, `draw_sprite`, particles, tilemap tilesets and `ImageWidget`.

Most games never touch it: `Sprite.texturePath` and friends load through it automatically. Reach for it when you need the loaded texture object itself, or need to swap the loader (tests, custom hosts).

---

## Use

```typescript
import { TextureStore } from '@emptysock/engine'

const store = new TextureStore()                 // default: the engine's normal asset loading

const already = store.get('./hero.png')          // the loaded texture, or undefined. Never starts a load.
const tex = await store.load('./hero.png')       // loads (or returns the cached one)
```

- `get(path)` is synchronous and only ever reads what is already loaded. It is what per-frame code should call.
- `load(path)` resolves from cache when it can. **Concurrent calls for one path share one load**, so calling it from many places at once is safe.
- A failed load is **not cached**: the next `load` retries.
- `clear()` drops the store's own local cache.

## Custom loader (tests and custom hosts)

```typescript
const store = new TextureStore(async (path) => makeTextureSomehow(path))
```

With a custom loader the store keeps its own small cache plus an in-flight table, so a path is still loaded once. Without one, it delegates to the engine's default asset cache and keeps no second copy.

Passing a loader to `RenderPipeline` (`textureLoader`) or the UI system (`imageLoader`) is how the engine's own systems get this seam; give them the same function and they behave identically.

## Rules

| Wrong | Right |
|---|---|
| Keeping your own `Map<string, Texture>` next to the engine's | Use `TextureStore`, so there is one cache to invalidate |
| Calling `load` every frame to check for readiness | Call `get` (cheap, synchronous) and `load` once |
| Assuming `clear()` frees the engine's global cache | It only drops the store's local one |
