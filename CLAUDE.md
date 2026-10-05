# CLAUDE.md

> **Deprecated / archived.** EmptySock development has stopped. This repo documents the engine's final state; keep edits minimal and only to correct inaccuracies.

Before editing any file, read it first. Before modifying a function, grep for all callers. Research before you edit.


This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## What this repo is

`emptysock-ai-skills` is a **documentation-only** companion to the EmptySock game engine. It contains no source code, no build tooling, and no tests. Every file in `skills/`, `docs/`, and `ai/` is Markdown or JSON that gets consumed by AI agents or human developers. The engine source lives in a separate private repository.

There are no build, lint, or test commands to run here — changes are purely content edits.

---

## Repository layout

| Path | Purpose |
|------|---------|
| `ai/CLAUDE.md` | Drop into a game project root for Claude Code — full engine rules |
| `ai/.cursorrules` | Drop into a game project root for Cursor / Windsurf |
| `ai/AGENTS.md` | Vendor-agnostic agent instructions (any AI tool) |
| `ai/api-reference.json` | Machine-readable full public API with types and examples |
| `skills/*.md` | Topic-scoped agent skill files covering individual engine systems |
| `docs/*.md` | Human-readable getting-started and concept guides |

---

## Versioning contract

This repo is frozen at the engine's final state (see `CHANGELOG.md`, final release 2026-10-05). Never document internal implementation details: only the public API the engine re-exports from `packages/*/src/index.ts` (and `packages/engine/dist-types`). A class that exists in engine source but is not re-exported must be described as internal, not as importable.

---

## Content rules

- **`ai/api-reference.json`** is the single source of truth for the public API. If a method appears there, it must not contradict what the skill files say. If a system is ever documented again, update `api-reference.json` first, then add the matching skill file.
- **`skills/` files** are loaded by agents at task time. Keep each file focused on one system. The `00-quickstart.md` file is the agent-facing cheat sheet — it should stay to one page.
- **Banned content** — never mention internal types, Rapier/PixiJS/Howler internals, or implementation files from the engine source. Agents must only import from `@emptysock/engine`.
- **Code examples** in skill files must follow the same rules as `ai/AGENTS.md`: no `any`, no `!`, no direct library imports, no `async onUpdate()`, no `setTimeout` in game logic. Every imported name must exist in the engine's current exports; verify with a grep over the engine index before citing it.

---

## Making changes

When editing skill files, also check whether `ai/CLAUDE.md`, `ai/AGENTS.md`, and `ai/api-reference.json` need to stay consistent. When a skill file describes a new API shape, update all four.

Any change that adds, renames or removes a skill file must also update the skills table in `README.md` so it matches `skills/` exactly (and its file count).
