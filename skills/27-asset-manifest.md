# AssetManifest

**Use this when** you need to preload textures, audio, JSON, or fonts up front with a progress bar and graceful per-asset failure handling, instead of hoping everything loads in time. The engine never renders a loading screen for you (see `skills/00-quickstart.md`) — build your own start scene and drive it off `AssetManifest.onProgress()`.

---

## Setup

```typescript
import { AssetManifest, AudioSystem } from '@emptysock/engine'

class LoadingScene extends Scene {
  override async onLoad(): Promise<void> {
    const manifest = new AssetManifest({ audioSystem: AudioSystem })

    manifest
      .add({ id: 'hero', path: 'assets/hero.png', type: 'texture' })
      .add({ id: 'jump_sfx', path: 'assets/jump.wav', type: 'audio' })
      .add({ id: 'level1', path: 'assets/level1.json', type: 'json' })

    const unsubscribe = manifest.onProgress((loaded, total) => {
      this._progressBar.value = loaded / total
    })

    const result = await manifest.load()
    unsubscribe()

    if (result.failed.length > 0) {
      for (const f of result.failed) console.warn(`failed to load ${f.id}: ${String(f.error)}`)
    }

    this.sceneManager.load('GameScene')
  }
}
```

By default (`continueOnError: true`) a failed asset is recorded in `result.failed` and loading continues; pass `continueOnError: false` to have `load()` reject on the first failure instead.

---

## Why it warms the real caches

Loading a texture through `AssetManifest` calls the same `textureLoader` `RenderPipeline` uses (`Assets.load` by default) — pass the exact loader you gave `RenderPipeline`'s `textureLoader` option so both hit the identical cache:

```typescript
const textureLoader = async (path: string) => myPipeline.resolve(path)
const render = new RenderPipeline({ textureLoader })
const manifest = new AssetManifest({ textureLoader })
```

Loading an `'audio'` descriptor calls `AudioSystem.load(id, path)` directly, registering it under `AudioSystem`'s own sound map — `AudioSystem.play(id)` afterwards hits the already-loaded sound with no redundant fetch. An `AssetManifest` constructed without `audioSystem` throws if it encounters an `'audio'` descriptor.

---

## Reading loaded assets back

```typescript
if (manifest.has('level1')) {
  const data = manifest.get('level1')   // unknown — narrow it yourself
}
```

`get()`/`has()` are most useful for `'json'`/`'font'` assets that have no other cache to land in — textures and audio are more naturally read back through `RenderPipeline`/`AudioSystem` themselves once loaded.

---

## Rules

| Wrong | Right |
|---|---|
| Building a bespoke `Promise.all` loader per project | Use `AssetManifest` — it already handles progress and per-asset failure |
| Assuming a failed asset aborts everything | By default it keeps going past failures; check `result.failed` / `manifest.failures` |
| Loading `'audio'` descriptors without passing `audioSystem` | `AssetManifest` throws for audio assets if there's no `AudioSystem` reference to hand them to |
| Expecting the engine to show a loading UI | It never does — read `onProgress()` and build your own scene |
