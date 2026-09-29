# Changelog

All notable changes to the EmptySock Agent Pack are documented here.
Versioned in lockstep with the engine. Format follows [Keep a Changelog](https://keepachangelog.com).

---

## [Unreleased]

Changes staged for the next release.

### Added

- Skills: `37-signal-bus`, `38-global-store`, `39-rain-effects`, `40-surfaces-blend-modes`, `41-bitmap-fonts`, `42-texture-store`
- `api-reference.json` entries: `SignalBus`, `GlobalStore`, `RainGlassFilter` (with `rainParticlePreset`), `GmlSurfaces`, `BitmapFontDef`, `TextureStore`

### Changed

- Remappable controls now live on `InputManager` (`skills/28`, `api-reference.json` `InputManager`); the separate `InputBindings` entry is removed
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
