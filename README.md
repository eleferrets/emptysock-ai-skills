# EmptySock Agent Pack

**Public documentation and AI agent reference for the [EmptySock](https://github.com/your-org/emptysock) game engine.**

This repository is the companion to the EmptySock engine. It contains everything needed to:

- Build games with EmptySock using AI agents (Claude Code, Cursor, Windsurf, and others)
- Learn the engine as a human developer
- Reference the full public API
- Understand how to export games to each platform

The engine itself is private. This repo is public and versioned in lockstep with it.

---

## Contents

```
emptysock-agent-pack/
  README.md                    ← You are here

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

| File | Description |
|------|-------------|
| `skills/00-quickstart.md` | Common patterns in one page — start here |
| `skills/01-actor-model.md` | Actor, ActorSystem, message-driven logic |
| `skills/02-navmesh.md` | PathfindingSystem (grid A*) and NavMeshSystem (polygon navmesh) |
| `skills/03-physics-3d.md` | PhysicsSystem3D (Rapier3D WASM): init, bodies, destroy |
| `skills/04-plugin-system.md` | PluginSystem singleton, inject, plugin lifecycle |
| `skills/05-touch-input.md` | Touch and pointer input via the static Input class |
| `skills/06-ide-panels.md` | IDE panel editors: Story Graph, Tilemap, Profiler, etc. |
| `skills/07-save-localisation.md` | SaveSystem slots and LocalisationSystem / t() API |
| `skills/08-story-graph.md` | Story Graph panel, VNSystem runtime, save/resume |
| `skills/09-particles.md` | ParticleEmitter: continuous and burst modes |
| `skills/10-window-system.md` | windowSystem: modes, size, title, compile-time constants |
| `skills/11-ui-system.md` | UISystem: animations, hover effects, render pass |
| `skills/12-vn-textbox.md` | VNTextbox and VNScriptConvert round-trip |
| `skills/13-variable-store.md` | VariableStore: named integer variables and boolean switches |
| `skills/14-character-stage.md` | CharacterStage, VNBackgroundLayer, VN render order |
| `skills/15-map-events.md` | MapEventSystem: tile triggers and command runner |
| `skills/16-battle-system.md` | Turn-based BattleSystem: party vs enemies, status effects |
| `skills/17-auto-tile.md` | AutoTileSystem: 8-neighbour bitmask rule sets |
| `skills/18-cg-gallery.md` | CGGallery: CG unlock tracking and display |
| `skills/19-tweens.md` | TweenManager: tweening with easing, scene-local timers |
| `skills/20-ui-widgets.md` | UISystem widgets: panels, labels, buttons, progress bars, sliders, checkboxes, animations |
| `skills/gms2-migration.md` | GMS2 → EmptySock migration: importer, GML mapping, asset status |
| `skills/layer-system.md` | LayerSystem: layer ordering and management |
| `skills/visual-script.md` | Visual Script Editor IDE panel: canvas controls, node types, .esvs format |

---

## Versioning

This repo is tagged to match engine releases: `v1.0.0`, `v1.1.0`, etc.

Always use the tag matching your installed engine version:

```bash
git clone https://github.com/your-org/emptysock-agent-pack
git checkout v1.0.0
```

Current stable: **v1.0.0**

---

## Quick links

- [Getting started →](docs/getting-started.md)
- [API reference →](ai/api-reference.json)
- [Agent quickstart →](skills/00-quickstart.md)

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
