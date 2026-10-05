# EmptySock Agent Pack

> **Deprecated.** EmptySock development has stopped. This pack documents the final state of the engine and is kept as a reference; it is no longer maintained. It is archived: do not expect updates.

**Public documentation and AI agent reference for the EmptySock game engine (final state).**

This repository is the companion to the EmptySock engine. It contains everything needed to:

- Build games with EmptySock using AI agents (Claude Code, Cursor, Windsurf, and others)
- Learn the engine as a human developer
- Reference the full public API
- Understand how to export games to each platform

This repo reflects the final state of the engine.

---

## Contents

```
emptysock-agent-pack/
  README.md                    ← You are here
  CHANGELOG.md                 ← Release history (final release 2026-10-05)
  CLAUDE.md                    ← Rules for editing this repo
  CONTRIBUTING.md              ← Archived; contribution notes

  ai/
    CLAUDE.md                  ← Drop in game project root for Claude Code
    .cursorrules               ← Drop in game project root for Cursor / Windsurf
    AGENTS.md                  ← Generic agent instructions (vendor-agnostic)
    api-reference.json         ← Machine-readable full API

  skills/
    (see Skills table below)

  docs/
    getting-started.md         ← Install EmptySock, first project, first game
    core-concepts.md           ← ECS, scene graph, game loop explained
```

---

## Skills

40 skill files, listed here exactly as they appear in `skills/`.

| File | Description |
|------|-------------|
| `skills/00-quickstart.md` | Common patterns in one page — start here |
| `skills/01-actor-model.md` | Actor, ActorSystem, message-driven logic |
| `skills/02-navmesh.md` | AStarSearch, Tilemap grids, and NavMeshSystem (polygon navmesh) |
| `skills/03-physics-3d.md` | PhysicsSystem3D (Rapier3D WASM): init, bodies, destroy |
| `skills/04-plugin-system.md` | PluginSystem (`ctx.plugins`), inject, plugin lifecycle |
| `skills/05-touch-input.md` | Keyboard, mouse, touch and pen input through InputManager |
| `skills/06-ide-panels.md` | IDE panel editors: Story Graph, Tilemap, Profiler, etc. |
| `skills/07-save-localisation.md` | SaveSystem slots and LocalisationSystem / t() API |
| `skills/08-story-graph.md` | Story Graph panel, VNSystem runtime, save/resume |
| `skills/09-particles.md` | ParticleEmitter: continuous and burst modes, mounting on a layer |
| `skills/10-window-system.md` | WindowSystem (`ctx.window`): modes, size, title, compile-time constants |
| `skills/11-ui-system.md` | UISystem and WidgetTree: entity-based widgets, layout, pointer dispatch |
| `skills/12-vn-textbox.md` | VNTextbox and VNBackgroundLayer |
| `skills/13-variable-store.md` | VariableStore: named integer variables and boolean switches (`ctx.variables`) |
| `skills/16-battle-system.md` | Turn-based BattleSystem: party vs enemies, status effects |
| `skills/17-auto-tile.md` | AutoTileSystem (@emptysock/tilemap): 8-neighbour bitmask rule sets |
| `skills/18-cg-gallery.md` | CGGallery: CG unlock tracking through a StorageAdapter |
| `skills/19-tweens.md` | TweenManager: tweening with easing, timers |
| `skills/20-ui-widgets.md` | UI widget patterns: panels, labels, buttons, progress bars, sliders, checkboxes |
| `skills/21-post-process.md` | PostProcessSystem: full-screen effects, per-layer filters, scene transitions |
| `skills/22-lighting-system.md` | LightSource / LightOccluder components and LightingSystem.collectLights |
| `skills/23-rendering.md` | RenderPipeline: Transform+Sprite auto-rendering, tilemap mounting, layers |
| `skills/24-viewport-system.md` | ViewportSystem: design-resolution scaling, safe-area insets, GPU-tier render defaults |
| `skills/25-pointer-system.md` | Pointer input: mouse/touch/pen, gestures, wheel/trackpad (via InputManager) |
| `skills/28-accessibility-debugging.md` | `InputManager` remappable actions and persistence, DebugOverlaySystem, text-scale pattern, and the colourblind simulation filter |
| `skills/29-visual-script-component.md` | Visual script runtime: VisualScriptState, VisualScriptSystem, VisualScriptGraphBuilder, ActorSystem bridging |
| `skills/30-custom-shader-filter.md` | CustomShaderFilter and the shader registry: custom GLSL filters and per-sprite shaders |
| `skills/31-sequence-system.md` | SequenceSystem: keyframe sequences played through TweenManager, plus pure scrubbing |
| `skills/32-ecs-core.md` | The entity/component/scene core: `defineComponent`, `Entity`, `Scene.each`, `Game` lifecycle, `ServiceRegistry` |
| `skills/33-save-system.md` | `SaveSystem`: generic component save/load, versioning, migrations, storage adapters |
| `skills/34-prefabs-pooling.md` | `definePrefab`, prop-override matching, pooling folded into spawn/destroy |
| `skills/35-network-package.md` | `@emptysock/network`: `networked()` field marking, `NetworkSystem`, `NetworkEntityMap` |
| `skills/36-visual-script-compiler.md` | The Visual Script compiler (`compileVisualScriptGraph`) |
| `skills/37-signal-bus.md` | `SignalBus`: game-wide named signals, `SignalGroup` cleanup, typed `GameSignals` |
| `skills/38-global-store.md` | `GlobalStore` and the typed `GameGlobals` interface (`game.globals`, `ctx.globals`) |
| `skills/39-rain-effects.md` | `RainGlassFilter` (`'rain-glass'` layer filter) and `rainParticlePreset` |
| `skills/41-bitmap-fonts.md` | `BitmapFontDef`, `FontRegistry.registerBitmap`, and bitmap-text layout helpers |
| `skills/42-texture-store.md` | `TextureStore`: the shared texture load/lookup path and its custom-loader seam |
| `skills/layer-system.md` | LayerSystem: layer ordering and management |
| `skills/visual-script.md` | Visual Script Editor IDE panel: Logic Script nodes and runtime hand-off |

---

## Versioning

This pack is frozen at the engine's final state. The last release is described in `CHANGELOG.md` ("Final release: deprecated, GML/GMS2 content removed", 2026-10-05). There are no further engine releases to track and no version tags to check out.

---

## Quick links

- [Getting started →](docs/getting-started.md)
- [API reference →](ai/api-reference.json)
- [Agent quickstart →](skills/00-quickstart.md)
- [Changelog →](CHANGELOG.md)

---

## Using with AI agents

| Agent / IDE | File to use | How |
|---|---|---|
| **Claude Code** | `ai/CLAUDE.md` | Copy to your game project root. Read automatically. |
| **Cursor** | `ai/.cursorrules` | Copy to your game project root. Read automatically. |
| **Windsurf** | `ai/.cursorrules` | Copy to your game project root. Read automatically. |
| **Claude.ai chat** | `skills/00-quickstart.md` + `ai/api-reference.json` | Upload at conversation start. |
| **Any agent** | `ai/AGENTS.md` | Use as a system prompt or context prepend. |

For deeper context on a specific task, pass the relevant skill file to the agent:

> *"Read the contents of skills/02-navmesh.md then implement a grid pathfinding enemy."*

---

## License

MIT. Free to use, fork, and redistribute.
The engine is separately licensed — see the engine repository.
