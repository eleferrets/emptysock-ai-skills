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
  CHANGELOG.md                 ← What changed per version
  CONTRIBUTING.md              ← How to report issues, request docs

  ai/
    CLAUDE.md                  ← Drop in game project root for Claude Code
    .cursorrules               ← Drop in game project root for Cursor / Windsurf
    AGENTS.md                  ← Generic agent instructions (vendor-agnostic)
    api-reference.json         ← Machine-readable full API

  skills/
    00-quickstart.md           ← Start here. Common patterns in one page.
    01-project-setup.md        ← New project, manifest, folder layout
    02-scenes.md               ← Scenes, lifecycle, navigation
    03-entities.md             ← Entities, components, ECS, prefabs, pools
    04-rendering.md            ← Sprites, animation, camera, lighting
    05-physics.md              ← Bodies, character controller, sensors, joints
    06-input.md                ← Keyboard, mouse, touch, gamepad, rumble
    07-audio.md                ← Playback, groups, spatial, sprites
    08-tilemaps.md             ← Loading, layers, collision, auto-tile
    09-particles.md            ← Emitters, parameters, performance
    10-ui.md                   ← Components, layout, anchors, accessibility
    08-story-graph.md          ← Story Graph panel, VNSystem, save/resume, localisation
    11-visual-novel.md         ← Dialogue trees, branching, auto-advance
    12-pathfinding.md          ← A* grid, navmesh, path following
    13-save-system.md          ← Slots, auto-save, settings persistence
    14-localisation.md         ← Translations, RTL, plural rules
    15-post-processing.md      ← Transitions, effects, colour grading
    16-coroutines.md           ← Generator-based async game logic
    17-tweens-timers.md        ← Tweens, timers, easing functions
    18-3d.md                   ← Hybrid 2D/3D scenes, Three.js layer
    19-export.md               ← All platforms, build pipeline, icons
    20-performance.md          ← GPU tiers, draw calls, Pi4, mobile
    21-typescript.md           ← Strict TS rules, Zod, banned patterns
    10-window-system.md        ← WindowSystem, window modes, compile-time constants

  docs/
    getting-started.md         ← Install EmptySock, first project, first game
    core-concepts.md           ← ECS, scene graph, game loop explained
    templates.md               ← Built-in templates and what they contain
    troubleshooting.md         ← Common errors and how to fix them
    migration.md               ← Upgrading between versions
    faq.md                     ← Frequently asked questions
```

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
- [Export guide →](skills/19-export.md)
- [Performance guide →](skills/20-performance.md)
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

> *"Read the contents of skills/05-physics.md then implement a platformer character with coyote time."*

---

## License

MIT. Free to use, fork, and redistribute.
The engine is separately licensed — see the engine repository.
