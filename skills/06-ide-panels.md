# IDE Panel Editors

The EmptySock IDE includes full editor panels accessible via the docked layout. All panels are dockable, re-arrangeable tabs.

## TilemapEditor
Canvas-based tile painter. Palette on the left, canvas on the right.

**Tools:** paintbrush, eraser, flood fill  
**Controls:** tile size slider, layer selector, zoom (scroll wheel)  
**Usage:** Open via `View > Tilemap Editor` or drag a `.tmj` asset into the dock.

## ParticleEditor
Live PixiJS preview alongside emitter property controls.

**Properties:** emission rate, speed (min/max), lifetime (min/max), gravity, start/end scale, start/end color (gradient picker), emission shape (point/circle/rect)  
**Usage:** Tweak values live; export settings as JSON for use with the engine's ParticleSystem.

## VNEditor (Visual Novel Node Graph)
SVG-based node graph for branching dialogue.

**Node types:** Dialogue (speaker + text), Choice (fan of option edges)  
**Interactions:** drag to move nodes, drag port to port to connect, double-click to edit text  
**Export:** JSON graph compatible with `VNSystem.loadScript()`.

## AudioMixer
Volume faders and mute/solo per audio bus group.

**Buses:** Master, Music, SFX, Voice, Ambient  
**Controls:** vertical fader (0–100%), mute toggle, solo toggle  
**Wired to:** `AudioSystem.setBusVolume(bus, volume)` in real-time.

## Profiler
Canvas frame-time bar chart with FPS history.

**Metrics:** frame time (ms), FPS, draw call count  
**Display:** 120-frame rolling history, color bands (green <16ms, yellow <33ms, red >33ms)  
**Usage:** Always visible in Play mode; detach to its own dock tab to keep it up while editing.

## LocalisationEditor
Spreadsheet-like table for managing translation strings.

**Columns:** key + one column per locale (e.g., en, fr, de, ja)  
**Editing:** inline click-to-edit cells  
**Import/Export:** CSV with header row `key,en,fr,...`  
**Usage:** Keys are looked up at runtime via `i18n.t('key')` (bring your own i18n library).

## GitPanel
Built-in lightweight Git commit helper.

**Shows:** modified/staged/untracked files (from Tauri shell commands)  
**Actions:** stage file, unstage, commit with message  
**Usage:** Requires `git` on PATH. Tauri desktop only — falls back to a stub in browser mode.
