# CustomShaderFilter

**Use this when** you're writing a custom GLSL post-process shader and need it to work identically in the IDE's **Shader Editor** preview and in the actual game. A shader written and previewed there compiles through the same `createCustomShaderFilter()` function, so there's exactly one shader-compile path, not a preview one and a "real" one that can drift apart.

---

## Uniform / attribute contract

Every custom shader must follow the PixiJS v8 filter contract:

- Attributes: `aPosition` (vec2), `aUV` (vec2)
- Vertex uniforms: `uProjectionMatrix`, `uWorldTransformMatrix`, `uTransformMatrix` (mat3) — standard PixiJS v8 filter uniforms
- Fragment: `uTexture` (sampler2D) — the filtered input
- Fragment: `uTime` (float) — seconds; you update it yourself via `setTime()`, the filter does not advance it on its own

Shaders are GLSL ES 3.00 style — `in`/`out`, not `attribute`/`varying`; `texture()`, not `texture2D()`.

A filter draws a full-screen quad, so a vertex stage that reads a per-vertex `aUV` and the three projection matrices (the sprite-quad shape above) is adapted for you: the engine keeps its `out` varyings and substitutes the filter's own positioning. You can keep writing the vertex stage in that shape, or omit `vertexSrc` entirely (the default is already correct). A vertex stage already written for a filter is used as written.

---

## Usage: one-off filter object

```typescript
import { createCustomShaderFilter } from '@emptysock/engine'

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
  // vertexSrc?: string        defaults to a correct full-screen filter vertex stage
  // uniforms?: Record<string, { value: number | number[]; type: string }>   extra scalar/vector uniforms, declared up front
  // wgslFragment?: string     WGSL fragment stage; without it the filter is GL-only and renders nothing under WebGPU
  // name?: string             debug label
})

// Each frame, only if the shader reads uTime:
filter.setTime(elapsedSeconds)
filter.setUniform('uStrength', 0.5)   // writes a declared uniform; ignored if the name was not declared
```

`createCustomShaderFilter` returns a `CustomShaderFilter`, which is a PixiJS `Filter`. Attach it by assigning to a PixiJS container's `filters` array, for example `render.stage.filters = [filter]` on a `RenderPipeline`'s stage (remove it by assigning `[]` or `null`). `RenderPipeline` has no per-layer shader-filter method.

## Usage: registered shader on sprites

For shaders shared across many sprites, register the source once and point `Sprite.shader` at the id; `RenderPipeline` builds one shared filter per id and applies it to those sprites:

```typescript
import { registerShader, setShaderUniform, Sprite } from '@emptysock/engine'

registerShader('sh_white', {
  vertexSrc: vertexGlsl,        // GLSL ES 3.00; only its `out` varyings are used, positioning is substituted
  fragmentSrc: fragmentGlsl,
  // wgslFragmentSrc?: string   optional, enables WebGPU
})

hero.add(Sprite, { texturePath: 'hero.png', shader: 'sh_white' })
setShaderUniform('sh_white', 'uAmount', 'f', [0.5])   // kind 'f' float / 'i' int; values array per component
```

Other registry helpers: `unregisterShader(id)`, `hasShader(id)`, `getShader(id)`, `shaderIds()`, `getShaderUniforms(id)`, `clearShaders()`. Uniforms are per shader id, shared by every sprite using it (a per-sprite value would need a separate shader id). Scalar and vector uniforms (`float`, `int`, `vec2..vec4`) declared in the fragment source are picked up automatically via `parseShaderUniforms`.

---

## Rules

| Wrong | Right |
|---|---|
| Writing `attribute`/`varying` or calling `texture2D()` | Use GLSL ES 3.00 style: `in`/`out` and `texture()` |
| Expecting `uTime` to advance on its own | Call `filter.setTime(elapsedSeconds)` yourself, every frame, if the shader reads it |
| Hand-wiring a filter onto a layer | `RenderSystem` (exported) has `addLayerShaderFilter(layerName, filter)`, but `RenderPipeline` owns its own instance; the simple routes are assigning the filter to a container's `filters`, or `registerShader` + `Sprite.shader` |
| Assuming a GL shader works under WebGPU | Supply `wgslFragment` (or `wgslFragmentSrc`) to support WebGPU; otherwise it is GL-only |
