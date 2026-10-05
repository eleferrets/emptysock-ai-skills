# Changelog

All notable changes to the EmptySock Agent Pack are documented here.
Versioned in lockstep with the engine. Format follows [Keep a Changelog](https://keepachangelog.com).

---

## [Final release] — 2026-10-05

Final release: deprecated, GML/GMS2 content removed.

EmptySock development has stopped; this pack is archived and documents the engine's final state.

### Removed

- All GML/GMS2-related content from skills, `api-reference.json` and docs
- Skills that documented APIs that no longer exist in the engine: `14-character-stage`, `15-map-events`, `26-animator-controller`, `27-asset-manifest`, and the surfaces/blend-modes skill
- `api-reference.json` entries for symbols the engine does not export: `MapEventSystem`, `GridMovementBehavior`, `CharacterStage`, `AnimatorController`, `AssetManifest`, `VisualScriptComponent`, and the legacy `Engine` / `SceneManager` / class-based `Scene` / `Timer` / `Audio` entries

### Changed

- All skills, `ai/*` and `docs/*` re-verified against the engine's public exports and rewritten where they had drifted: scenes are `defineScene` objects (not classes), UI is entity-based (`WidgetTree`, `UISystem`, widget components), input goes through `InputManager`, lighting is `LightSource` / `LightOccluder` components, and `VisualScriptSystem` replaces `VisualScriptComponent`
- `api-reference.json` rebuilt from the current exports (spot-checked against engine signatures); the raw system classes (`InputSystem`, `PointerSystem`, `GamepadSystem`, `AudioSystem`, `PostProcessSystem`, `ParticleSystem`, `CoroutineSystem`, `RenderSystem`) and the `accessibilitySettings` singleton are now exported by the engine and documented as such
- README skill index regenerated to match `skills/` exactly (40 files); deprecation banners added to `ai/*` and `docs/*`

### Added

- `RenderPipeline.attachLighting(lighting, layerId?)` is documented in `skills/22-lighting-system.md` and `api-reference.json`: lighting now renders through the pipeline instead of requiring a hand-driven `RenderSystem`
- `skills/36-visual-script-compiler.md` and `skills/29-visual-script-component.md` now describe `compileVisualScriptGraph`, `VisualScriptState` and `VisualScriptSystem`

---

## [2.0.0] — earlier

### Added

- Skills: `37-signal-bus`, `38-global-store`, `39-rain-effects`, `41-bitmap-fonts`, `42-texture-store`
- `api-reference.json` entries: `SignalBus`, `GlobalStore`, `RainGlassFilter` (with `rainParticlePreset`), `BitmapFontDef`, `TextureStore`

### Changed

- Remappable controls moved onto `InputManager` (`skills/28`, `api-reference.json` `InputManager`); the separate `InputBindings` entry was removed
- `'rain-glass'` added to the layer filter types; `CustomShaderFilter` documents that sprite-style vertex stages are adapted automatically

---

## [1.0.0] — TBD

Initial public release.

### Added

- Full AI agent reference: `CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `api-reference.json`
- 21 skill files covering every engine system
- Human documentation: getting started, core concepts, templates, troubleshooting, FAQ
- Migration guide structure (empty until v1.1)
- Public API reference (all systems, all methods, typed signatures)

---

*This file is updated with every engine release.*
