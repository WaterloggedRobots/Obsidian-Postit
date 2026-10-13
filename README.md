# Canvas Color & Style

An Obsidian plugin that extends the built-in **Canvas** feature with:

- **More colors, everywhere** — a "More colors" submenu (editable in Settings) on
  both cards and connecting lines, on top of Obsidian's own 6 presets, plus a
  "Custom color…" picker for any hex value.
- **Line style** — solid / dashed / dotted, and a straight-line option alongside
  Canvas's default curve.
- **Quick-select panel** — a side pane (ribbon icon or command palette: "Open canvas
  quick-select panel") listing every card on the open canvas by its first line of
  text, colored by its card color. Click a row to select that card and jump the
  camera to it — the same "click a note in the list to fly to it" panel from the
  original sticky-note desktop app this was ported from.

## Install (from source)

1. `npm install`
2. `npm run build`
3. Copy `manifest.json`, `main.js`, and `styles.css` into
   `<your-vault>/.obsidian/plugins/canvas-color-style/`
4. In Obsidian: Settings → Community plugins → enable **Canvas Color & Style**.

For active development, `npm run dev` watches and rebuilds `main.js` on save.

## How it's built, and what's solid vs. fragile

Obsidian's Canvas has no official public API for extending it, so every canvas
plugin (this one included) is working with some mix of documented-but-unofficial
hooks and outright internals. This plugin is deliberately split so the risk is
contained to one file:

| Feature | Mechanism | Stability |
|---|---|---|
| Card & line colors | The `canvas:node-menu` / `canvas:edge-menu` workspace events Canvas fires on right-click, plus the `color` field every card/line already stores (a hex string — no schema change needed) | Solid. These events are used by several community canvas plugins and are unlikely to disappear. |
| Custom color picker | A plain Obsidian `Modal` wrapping a native `<input type="color">` | Solid — no canvas internals involved at all. |
| Quick-select panel | Reads `canvas.nodes` (a live read, not a patch) and calls `canvas.selectOnly()` / `canvas.zoomToSelection()` if present | Mostly solid. The read is safe; the two navigation calls are wrapped individually, so if a future Obsidian version renames them, clicking a row just stops animating the jump instead of throwing. |
| Dashed / dotted / straight lines | **The one real internals patch.** Stores two extra fields on each edge's data, tags the rendered `<path>` with matching `data-*` attributes via a small `monkey-around` patch, and lets `styles.css` do the actual dashing | Fragile by nature — see below. |

`src/edgeStyle.ts` has a large comment at the top explaining exactly why this piece
is different from the rest, and it's the only thing gated by a settings toggle
("Enable dashed / dotted / straight lines"). If an Obsidian update ever breaks line
styling, turn that toggle off — colors and the quick-select panel are unaffected,
since they don't go through that code path at all.

One more note on typings: `src/canvasCompat.ts` exists solely to hold the handful of
`as any` casts this plugin genuinely needs — Obsidian's shipped types only declare
`Workspace.on()` overloads for Obsidian's own built-in events (Canvas's
`canvas:node-menu` / `canvas:edge-menu` aren't in that list, despite firing for
real), and `MenuItem.setSubmenu()` isn't in the public `.d.ts` either, despite being
how Canvas builds its own nested menus. Both are real, working APIs — just
undocumented ones — so the casts live in one small, clearly-commented file instead
of being scattered through the rest of the source.

If you'd rather not maintain this kind of internals patch yourself long-term, the
community plugin **[Advanced Canvas](https://github.com/Developer-Mike/obsidian-advanced-canvas)**
already implements a much more complete version of the line-style and node-shape
side of this (dotted/dashed/long-dashed edges, straight/A* pathfinding, border
styles, flowchart shapes) and is actively maintained — worth a look if you outgrow
what's here.

## Mobile (iOS / Android)

The plugin runs on Obsidian mobile. Touch-specific behavior:

- **Styling cards/lines** — long-press a card or line if your Obsidian version shows
  a menu, or select item(s) and run **"Color / style selected canvas item(s)"**.
  Add that command to the mobile toolbar (Settings → Mobile) for one-tap access.
  With several items selected, the chosen color is applied to all of them.
- **Submenus** (Quick colors, Line style, Line shape) open as a full-size picker sheet.
- **Polyline turning points** — use *Line shape → Add turning point* (no
  double-click on touch); tap a turning point to set its radius or remove it.
  Handles are drawn larger on mobile.
- **Quick-select panel** — tapping a card closes the drawer and jumps to it.

## Settings

- **Enable dashed / dotted / straight lines** — toggles the internals patch above.
- **Custom colors** — add, rename, recolor, or remove entries in the palette shown
  in the "More colors" submenu. Stored per-vault via Obsidian's plugin data, not in
  the `.canvas` files themselves.

## Notes on the data format

Line style and path style are stored as two extra fields directly on each edge's
JSON (`ccStyleLine`, `ccStylePath`). Obsidian's Canvas format tolerates unknown
fields — they round-trip through save/load untouched — but another canvas plugin
that aggressively strips unknown edge fields could theoretically drop them. Card
and line colors use Canvas's own native `color` field, so they're unaffected by
this either way.
