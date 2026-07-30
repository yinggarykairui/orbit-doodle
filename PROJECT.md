# PROJECT.md — orbit-doodle

Per MANUAL §4 (PROJECT.md rule): the converged spec, architecture, done-map,
and open threads. Backfilled on first revisit (day 006) — this is a repo that
shipped (day 004) before the rule reached it. Revisits read this first and
update it last; a revisit's planner diffs repo state against this spec and
specs only the next increment.

## Spec (converged)

**v0 (day 004, seeded idea).** A single-file canvas drawing toy where the
visible pen orbits an invisible center, and that center chases the pointer
with lerp momentum. Steering is indirect on purpose: fast drags stretch the
orbit into long loops, slow drags coil it into knots. Stroke width tapers
with pen speed (thin when fast, thick when slow) and is smoothed so it never
pops. All physics constants are expressed per 60 fps frame and normalized by
`dt`, so the feel is frame-rate independent. Five-colour palette, Clear, Save
PNG. Background is painted onto the canvas so exports are WYSIWYG. Canvas
fills the viewport above a wrapping control bar; no page scroll ever. Backing
store sized for `devicePixelRatio`, context kept in CSS-pixel units.

**Increment 2 (day 006 revisit — stroke history and three pens).** The
canvas stops being the only record of the drawing. Every stroke is recorded
as a replayable path and the drawing is *redrawn from that history* whenever
it needs to change — undo, redo, clear, resize. On top of that history, a
pen picker: three parameter sets whose orbit physics produce visibly
different lines, carried on the stroke so replay is faithful.

In scope:

- **Stroke history.** A linear history of entries with a cursor. Entry kinds:
  a stroke (pen id, colour, and the sampled pen path) or a clear marker.
  Undo moves the cursor back one entry, redo forward one; drawing a new
  stroke truncates anything past the cursor and appends. Redraw = repaint the
  background, then replay entries `0..cursor`.
- **Undo/redo controls.** Two buttons in the control bar (`↶` / `↷`, 40 px
  targets, `aria-label` undo/redo), visibly disabled when there is nothing to
  undo or redo. Keyboard: Ctrl/Cmd+Z undoes; Ctrl/Cmd+Shift+Z and Ctrl+Y redo.
  Buttons and shortcuts drive the same code path.
- **Clear becomes one undoable step.** Clear appends a clear marker rather
  than destroying state; Undo after Clear restores the whole drawing. Clear on
  an already-clear canvas appends nothing (no dead undo steps).
- **Three pens.** `orbit` (the day-004 pen, unchanged constants — existing
  work must still be reproducible in feel), `coil` (small radius, high angular
  velocity, snappier chase, finer line — a tight spring), `drift` (large
  radius, low angular velocity, laggier chase, heavier line — long lazy
  loops). Picker sits in the control bar with the same active-state styling as
  the colour swatches. A pen applies to strokes drawn after it is picked;
  switching pens never restyles earlier strokes.
- **Deterministic redraw on resize** (planner's judgement call, in): resize
  re-sizes the canvas and replays history instead of stretching a bitmap
  snapshot. This *deletes* the snapshot-restore block rather than adding code,
  and it is forced by correctness — once undo replays from history, the
  snapshot path is a second, disagreeing source of truth for what is on the
  canvas. Side benefit: content pushed off a narrowed canvas comes back when
  the window widens, instead of being cropped away permanently.

**Fence (excluded — a future spec comment must open these):** persistence of
any kind, including localStorage · share links (pixel-garden owns that
mechanic) · export changes beyond keeping Save PNG working exactly as it does
· any new dependency, file, or build step (stays one vanilla HTML file, §13)
· palette changes (still the same five colours) · animation, replay scrubbing,
or any timeline UI · layers · eraser · brush-size or physics sliders · stroke
selection or per-stroke editing · a history panel.

## Architecture sketch

Shipped build (day 006): one IIFE in `index.html`, no globals, no deps, no
build step. Three parts:

- **Layout/CSS.** Flex column: `#stage` (canvas + DOM hint overlay) over
  `#controls`. The hint lives in the DOM, not the canvas, so it never lands in
  an export. Controls wrap, every target ≥ 40 px.
- **Physics loop.** A permanent `requestAnimationFrame` tick. While `drawing`,
  the orbit center lerps toward the pointer (`alpha = 1 - (1-CHASE)^frames`),
  the angle advances, and the segment from the previous pen position to the
  new one is stroked directly to the context. `dt` is clamped to 3 frames so a
  backgrounded tab cannot produce one giant jump. `firstFrame` suppresses the
  teleport segment right after `pointerdown`.
- **Chrome.** Swatches built from the `PALETTE` array (colour data lives in
  one place); Clear repaints the full backing store in device pixels; Save PNG
  goes through `toBlob` with a null guard; resize is debounced 100 ms and
  replays history (the day-004 snapshot-restore is gone).

Increment 2 changed the shape as follows, and this is what ships today:

- **History is the source of truth; the canvas is a view of it.** Any code
  that changes what is on screen goes through one `redraw()` that repaints the
  background and replays entries up to the cursor. Nothing else clears or
  bulk-paints the canvas.
- **Record the pen path, not the pointer path.** A stroke stores the pen
  samples the physics emitted (`x`, `y`, smoothed width per sample), not raw
  pointer input. Re-simulating from pointer points would depend on frame
  timing, so undo/redo/resize would each redraw a slightly different picture;
  storing the emitted path makes replay exact and keeps the physics code
  single-purpose. Cost is a few hundred floats per stroke — nowhere near the
  ~10 MB a dpr-2 full-canvas bitmap would cost per undo step.
- **Live drawing stays incremental.** The in-progress stroke is still stroked
  segment-by-segment as it is drawn (no replay per frame); its samples are
  appended to a pending stroke that is committed to history on pointer-up.
- **A pen is a parameter set.** `PENS = { orbit, coil, drift }`, each with
  chase, angular velocity, orbit radius, and width min/max. The live loop
  reads the active pen; a stroke carries its pen id so replay reads the same
  set. The day-004 constants become the `orbit` entry verbatim.
- **Resize** became `sizeCanvas(); redraw();` — the debounce stays and the
  offscreen snapshot is gone. The day-004 zero-size guard went with it: it
  existed only so `drawImage` could not throw on a zero-sized snapshot, and
  `fillRect`/`stroke` at zero size are no-ops (confirmed under a rapid-resize
  storm).

## Done-map

**v0 (day 004)** — complete, shipped:

- [x] Orbit pen physics: lerp center-chase, angular advance, `dt`-normalized
      constants, clamped `dt`, no teleport segment on pointer-down
- [x] Speed-tapered, smoothed stroke width
- [x] Five-colour palette with swatch picker built from one data source
- [x] Clear (full backing store) and Save PNG via `toBlob` with null guard
- [x] Pointer capture, primary-button-only, blur/cancel release
- [x] Canvas sized for devicePixelRatio; resize survival via snapshot restore
- [x] Phone width: controls wrap at 375 px, targets ≥ 40 px, no page scroll
- [x] README + LICENSE + screenshot · Pages live · shipped day 004,
      rubric must-pass 7/7, avg 4.50

**Increment 2 (day 006)** — stroke history, undo/redo, three pens:

- [x] History model: entries (stroke | clear) + cursor; `redraw()` is the only
      path that repaints the canvas
- [x] Strokes record pen samples (x, y, width) + colour + pen id; no bitmap
      snapshots anywhere in the history path
- [x] Undo/redo buttons (`↶` `↷`, disabled when unavailable) and Ctrl/Cmd+Z,
      Ctrl/Cmd+Shift+Z, Ctrl+Y — same code path
- [x] New stroke after an undo truncates the redo tail
- [x] Clear is one undoable entry; no-op on an empty canvas
- [x] Resize replays history instead of stretching a snapshot; snapshot block
      removed
- [x] Pen picker: `orbit` (unchanged), `coil`, `drift`; active-state styling;
      pen carried per stroke through replay
- [x] Phone width holds with the enlarged control bar; Save PNG unchanged
- [x] README made true (already rewritten README-first), screenshot refreshed
      from the running build

## Open threads

- **Viewport-scaled orbit radius.** Not in the increment spec; added during the
  fix cycle because `drift`'s 95 px radius amputated strokes drawn within ~95 px
  of any edge on a 375 px canvas — the flagship pen silently ate the line. Pen
  radius, speed base and taper now scale by `min(1, shortViewportEdge / 640)`,
  so `orbit` is bit-for-bit the day-004 pen at any short edge ≥ 640 px
  (measured: 1440x900 and 1440x700 both 1.000) and shrinks below it
  (1440x620 0.969, 1280x600 0.938, 375x667 0.586). PROJECT.md's
  "`orbit` (unchanged constants)" holds above the threshold only. Whether a
  short desktop window should scale at all — it measures the viewport, control
  bar included, not the canvas — is the open question.

- **Unbounded history.** Nothing caps the entry list. A marathon session grows
  memory and slows replay linearly. The increment ships with a measured bound
  (undo after 50 strokes redraws well under 100 ms); a cap or a bake-the-tail
  strategy is a later call, and it interacts with undo depth, so it wants its
  own issue.
- **Resize is lossy in one direction only.** Replay preserves strokes that a
  narrowed window pushes off-canvas, so widening restores them; it does not
  reflow or rescale artwork to the new aspect. Reflow would change what the
  drawing *is*, and is out of scope.
- **Fence candidates a future owner issue could open:** an eraser (a stroke
  kind that composites `destination-out` replays cleanly in this model),
  per-pen colour memory, pressure/tilt input from a stylus, and pen parameter
  sliders for people who want to build a fourth pen.
- Save PNG exports the current view, which after this increment is exactly the
  replayed history — the WYSIWYG contract from v0 is unchanged, and should
  stay a checked assumption on every revisit.
