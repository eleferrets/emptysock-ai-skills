# Release Pass — Running Log

Active branch: `claude/nifty-fermat-wsmwml`
Session: https://claude.ai/code/session_01UY8VuKQkGwEmRFAMiP71kK

This file is the persistent task log for the pre-release quality pass.
Cross-repo source of truth is in `emptysock-engine/RELEASE_PASS.md`.
Update this file after any fix in this repo.

---

## Status key

- [x] Fixed and pushed
- [ ] Open / not started
- [~] Partial / needs verification

---

## Fixes in this repo (`emptysock-ai-skills`)

- [x] **`ai/AGENTS.md`** — Timer class → TweenManager instance; Audio → AudioSystem; Camera → instanced CameraSystem; removed waitForEvent; SaveSystem → instanced synchronous.
- [x] **`ai/CLAUDE.md`** — Camera section rewritten as instanced CameraSystem; coroutine example shake() fixed; SaveSystem made synchronous and instanced; common mistakes table corrected.
- [x] **`ai/api-reference.json`** — CameraSystem rewritten; GamepadSystem gets isButtonPressed/Released/Down; Widget.on() return type fixed; SaveSystem rewritten as instanced synchronous.
- [x] **`skills/00-quickstart.md`** — Replaced wrong static Camera API; added Gamepad section.
- [x] **`skills/01-actor-model.md`** — Added inbox cap (1000 msgs) and flush-pass re-send caveat.

- [x] **`ai/CLAUDE.md`** — "Common Mistakes" table removed; every entry is now a TypeScript compile error or ESLint rule.

## Remaining open items

None identified. See `emptysock-engine/RELEASE_PASS.md` for full cross-repo list.
