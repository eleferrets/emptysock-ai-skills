# CustomShaderFilter

A user-authored GLSL post-process filter, and the runtime counterpart to the IDE's **Shader Editor** panel. A shader written and previewed there compiles through the same `createCustomShaderFilter()` function, so the panel's preview and a game's actual render use one shader-compile path, not two.

---

## Uniform / attribute contract

Every custom shader must match the same contract `LightingSystem`'s built-in filter uses:

- Attributes: `aPosition` (vec2), `aUV` (vec2)
- Vertex uniforms: `uProjectionMatrix`, `uWorldTransformMatrix`, `uTransformMatrix` (mat3) — standard PixiJS v8 filter uniforms
- Fragment: `uTexture` (sampler2D) — the filtered input
- Fragment: `uTime` (float) — seconds; you update it yourself via `setTime()`, the filter does not advance it on its own

Shaders are GLSL ES 3.00 style — `in`/`out`, not `attribute`/`varying`; `texture()`, not `texture2D()`.

---

## Usage

```typescript
import { createCustomShaderFilter, RenderSystem } from '@emptysock/engine'

const filter = createCustomShaderFilter({
  fragmentSrc: `
    precision mediump float;
    in vec2 vUV;
    out vec4 finalColor;
    uniform sampler2D uTexture;
    uniform float uTime;
    void main() {
      finalColor = texture(uTexture, vUV + vec2(sin(uTime) * 0.01, 0.0));
    }
  `,
  // vertexSrc?: string — defaults to DEFAULT_CUSTOM_SHADER_VERTEX
})

// RenderSystem (not RenderPipeline) owns layer shader filters:
private _renderSystem = new RenderSystem()

override onLoad(): void {
  // ... this._renderSystem.init(...) as part of your render setup ...
  this._renderSystem.addLayerShaderFilter('default', filter)
}

override onUpdate(dt: number): void {
  this._elapsed += dt
  filter.setTime(this._elapsed)   // only needed if the shader reads uTime
}

override onDestroy(): void {
  this._renderSystem.removeLayerShaderFilter('default', filter)
}
```

`RenderSystem.addLayerShaderFilter(layerName, filter)` / `removeLayerShaderFilter(layerName, filter)` attach and detach the filter on a named render layer's PixiJS container. `RenderPipeline` (the batteries-included renderer most scenes use — see `skills/23-rendering.md`) does not forward these methods; a scene that needs a custom shader filter drives `RenderSystem` directly instead of, or alongside, `RenderPipeline`.

---

## Rules

| Wrong | Right |
|---|---|
| Writing `attribute`/`varying` or calling `texture2D()` | Use GLSL ES 3.00 style: `in`/`out` and `texture()` |
| Expecting `uTime` to advance on its own | Call `filter.setTime(elapsedSeconds)` yourself, every frame, if the shader reads it |
| Calling `addLayerShaderFilter` on a `RenderPipeline` instance | It lives on `RenderSystem` — `RenderPipeline` does not expose it |
| Forgetting to name a real layer | `addLayerShaderFilter`/`removeLayerShaderFilter` take the same layer names `LayerSystem.defineLayer()` created; `'default'` always exists |
