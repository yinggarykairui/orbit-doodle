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
- **A release is not the end of the stroke** (evening polish, day 006). The
  orbit center trails the pointer, so stopping the physics at `pointerup` threw
  away every pixel of that lag — `drift` ended 419 px behind a 1000 px drag.
  Pointer-up now makes the release point the target and keeps integrating into
  the *same* pending stroke until the center is within 0.5 px or a 900 ms cap
  expires; the tail is part of the one history entry, so undo, redo and replay
  stay faithful, and a new pointer-down commits it and takes over. An
  interruption (`pointercancel`, blur) is not a release and still ends flat.
- **A pen is a parameter set.** `PENS = { orbit, coil, drift }`, each with
  chase, angular velocity, orbit radius, and width min/max. The live loop
  reads the active pen; a stroke carries its pen id so replay reads the same
  set. The day-004 constants become the `orbit` entry verbatim.
- **Resize** became `sizeCanvas(); redraw();` — the debounce stays and the
  offscreen snapshot is gone. The day-004 zero-size guard went with it: it
  existed only so `drawImage` could not throw on a zero-sized snapshot, and
  `fillRect`/`stroke` at zero size are no-ops (confirmed under a rapid-resize
  storm).
- **The chrome asks the canvas, not the history** (evening polish, day 006).
  `hasInk()` used to count applied entries, which is a different question from
  "is anything on screen": a narrowed window pushes art off the edge without
  losing it, so the canvas could be visibly empty while the bar insisted it was
  not. It now tests whether any stroke sample lands inside `viewW × viewH`, a
  fourth hint names that state, and the four action buttons run off one
  `updateChrome`: the arrows follow the cursor, Clear and Save PNG follow
  visible ink (an enabled Clear with nothing to clear was a silent no-op, and
  Save PNG exported a blank rectangle).

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

**Evening polish (day 006)** — defects closed against the shipped increment,
no new scope. Every line here was checked by driving the build, not by reading
the diff; the off-canvas line below is the reminder that driving one state at a
time is not the same as driving the build. It passed a narrowed window with ink
in history and failed the same window after a Clear, and a third pass had to
close it. Lines a later pass reopened say so.

- [x] Provenance footer says day 006 (it said 004)
- [x] A released stroke finishes its tail instead of freezing behind the hand.
      Rightmost ink against a 200→1200 px drag at 1440x900: `drift` 419 px
      short → 16 px short, `orbit` 194 → 41 px past, `coil` 69 → 14 px past
      (the tail ends on the orbit circle, so "past" is the radius). One history
      entry, interruptible by the next pointer-down.
- [x] Pen scale measures the canvas, plus a radius cap of a twelfth of its
      short edge; phone edge-loss 38% → 20%; `orbit`/`coil` 0-pixel diff
- [x] `:hover` on all twelve controls (0 px → 176–3698 px each) and a
      `:focus-visible` outline distinct from the selected ring. The selected
      swatch's halo had no blur, so it resolved to a flat rgb(92,90,87) collar
      butted against the warm-white ring; blurred on the third pass, the same
      scan line ramps 27 → 67 over 10 px instead of stepping 27 → 92 → 245
- [x] `aria-pressed` on both toggle groups, set from the same place as the
      ring; swatches labelled by colour name, not `color 3`
- [x] Clear and Save PNG disable with the arrows, on visible ink
- [~] Save PNG acknowledges in-page (label + `aria-live`), and the ack is
      transient text rather than a rename: the button keeps a fixed
      `aria-label`. The third pass also meant to drop the ack when the canvas
      empties inside the 1.6 s window; that half is **not** closed, and the
      line here claiming it was has been corrected. See the open thread below —
      a fourth independent verifier reproduced the dimmed `Saved ✓`.
- [x] Chrome tells the truth when the art is off-canvas — **and only when it
      is** (reopened and closed on the third pass). The first version asked the
      question of every entry in the list, so an emptiness that Undo or Clear
      owned was blamed on the window, and that branch is tested first, so it
      won. Measured at 1440x900: draw near the right edge, Clear, narrow to
      500 px and the hint read `off-canvas — widen the window`; widening back
      to 1440 left 0 inked pixels and flipped it to `cleared`. With Undo the
      hint asked for a wider window while `↶` was disabled and `↷` — the
      control that helps — sat enabled and unnamed. The test now walks
      `visibleFrom() … cursor`, the span `redraw()` paints, so Clear reads
      `cleared` at 500 px and 1440 px alike, Undo reads `undone`, and a stroke
      genuinely outside the canvas still reads `off-canvas`. Re-driven at
      1440x900 and by rotating a 375x667 phone to landscape.
- [x] Background fills the backing store, so a fractional dpr no longer exports
      a half-transparent edge column (measured RGBA 16,16,16,128 → none)
- [x] Controls opt out of text selection and the touch callout
- [x] `<head>`: description, theme-color, Open Graph/Twitter, inline SVG
      favicon; load is still exactly 1 request
- [x] README's "What it does" run-on split, still 5 sentences
- [x] The `replayStroke` comment states the measured cost instead of claiming
      it is fast enough
- [x] `screenshot.png` recaptured from *this* build (third pass), 2400x1600 at
      deviceScaleFactor 2. The committed one predated the radius cap and the
      settle: the same `drift` drag at 1200x800 traces a 192 px loop on the
      pre-cap build and 119 px now, colour-keying gold gives a max loop outer
      span of 139.7 → 96.2 css px between the two PNGs, and every stroke in the
      old shot ended in a blunt round cap. Its caption — an italic paragraph
      between the image and the demo link, a line STYLE.md's template does not
      have — is now the image's alt text, which keeps the words and the
      template order both

## Open threads

- **Canvas-scaled orbit radius, and the edge loss it does not remove.** Not in
  the increment spec; added during the fix cycle because `drift`'s 95 px radius
  amputated strokes drawn near any edge of a 375 px canvas — the flagship pen
  silently ate the line. The scale first measured the *viewport*, which counts
  the control bar as drawing surface; the evening polish moved it to the canvas
  (`viewW`/`viewH`) and added a second limit, a radius cap of a twelfth of the
  canvas's short edge, so no pen's loop is wider than a sixth of it. `orbit` and
  `coil` are still bit-for-bit the day-004 pens wherever the scale is 1 —
  verified as a 0-pixel diff of the drag phase against the pre-polish build at
  1440x900, 1440x700 and 1024x768 — but the cap does bind on `drift` at desktop
  size (70.2 px rather than 95 px at 1440x900), because a 190 px loop clips
  against a desktop edge too, just less often. `screenshot.png` was recaptured
  from the capped build on the third polish pass, so it no longer advertises the
  wider `drift`.
  What is *not* closed: an orbiting pen drawn close to an edge still loses the
  arc that falls outside. On a 375x510 canvas, the same stroke drawn 25 px from
  the top now loses 20% of its ink against one drawn at centre, down from 38%,
  and the residual is inherent — driving it to zero means either a radius near
  25 px (which makes `drift` indistinguishable from `orbit` on a phone) or
  clamping the orbit center away from the edges, which would stop the ink
  landing where the finger points. Both are design calls for an owner issue,
  not a fix cycle.

- **`Saved ✓` can still land on a disabled button — a race, not a state bug.**
  Found by the fourth independent pass of the 2026-07-30 evening shift, after
  the loop cap was spent, so it is recorded rather than fixed. `ackSave()` runs
  inside the asynchronous `canvas.toBlob` callback, while the guard that drops
  the ack runs synchronously from `updateChrome()`. If the canvas empties during
  the blob encode — measured at 47.4 ms at 1440x900/dpr 2, 20–21 ms at dpr 1 and
  at 375x667 — the disable lands first and the ack lands after it, leaving a
  dimmed `Saved ✓` for the rest of the 1.6 s timer. Reproduces at delays of
  0–20 ms and reliably from `save.click(); clear.click();` in one task; clean at
  40 ms. No state corruption: the exported PNG is correct (`toBlob` snapshots at
  call time), the `aria-label` stays `Save PNG`, and it self-heals on the timer.
  The fix is to re-ask the ink question inside the callback rather than only
  before it.

- **Two chrome nits from the same pass.** The `off-canvas — widen the window`
  hint is reachable on a phone (320x568, or rotating 375x667 with a right-edge
  stroke) where widening is not an affordance the device has — rotating is the
  real remedy, and the copy should say something a phone can act on. And a
  control that disables itself under the keyboard drops focus to `<body>`, so
  the next Tab restarts at the first swatch; the pattern predates this shift on
  the arrows and now extends to Clear and Save PNG.

- **The three pens compress into one and a half on a phone.** Loop diameters,
  one slow horizontal drag per pen, widest column of ink: at 375x667 (canvas
  375x510) `coil` 19 px, `orbit` 53 px, `drift` 69 px; at 1440x900 (canvas
  1440x843) `coil` 30 px, `orbit` 85 px, `drift` 139 px. The gap between the
  two big pens narrows from 1.64x to 1.30x, so the flagship pen and the default
  pen draw nearly the same line on the device most people will meet the toy on.
  The cause is the two limits in `scaledPen` swapping which one is in charge:
  below the 640 px reference edge `orbit`'s radius comes from `penScale`
  (40/640 = 0.0625 × short edge) while `drift`'s is pinned by the cap
  (1/12 = 0.0833 × short edge), which locks their radius ratio at 1.33x; above
  it `penScale` saturates at 1 and the ratio opens back out to 1.76x at an
  843 px short edge. Line widths are identical at every size — `wMin`/`wMax` do
  not scale — which adds the same constant to both spans and flattens the drawn
  ratio a little further still. Not fixed here: every way out changes what a pen
  *is* on a phone (a smaller cap fraction, a `wMax` that scales with the radius,
  or a per-pen reference edge), and that is a design call with its own issue.

- **First load shows nothing the toy makes.** A stranger who opens the page gets
  a black rectangle, a control bar with two disabled arrows, and one line of
  text — `press and drag — the pen orbits you`. Nothing on screen demonstrates
  the flourish that is the entire point, so the toy asks for a gesture before it
  has shown what a gesture buys. `screenshot.png` and the demo link carry that
  job today, which works from the README and not at all from the URL. Not fixed
  here: every answer is a feature — a seeded demo stroke, a ghost animation, a
  first-run replay — and each adds state or motion the fence excludes. Its issue
  has to settle whether the demo is undoable, whether it counts as ink for Clear
  and Save PNG, and how it gets out of the way on the first pointerdown.

- **Unbounded history, and replay is not free.** Nothing caps the entry list,
  and a redraw is every segment drawn since the last clear — so undo scales
  with how much has been drawn, not with how deep the undo stack is. Measured
  at 1440x900 with `drift`, one-second drags, median of 5, canvas flushed
  inside the timing: 25 s of pen-down costs 17.7 ms per undo, 50 s 39.6 ms,
  100 s 71.7 ms, 150 s 105.5 ms — linear, ≈0.70 ms per second of pen-down. A
  resize pays the same replay after its 100 ms debounce: 225 ms end to end at
  150 s. So an undo stops being a frame's work at around a minute and a half of
  continuous drawing, and the earlier "well under 100 ms after 50 strokes" note
  was true only for the stroke lengths it was measured with. The settle tail
  added by the evening polish makes each released stroke record more samples,
  which moves a session up this curve faster; per second of recorded pen-down
  the rate is unchanged. Not fixed here: the batched-polyline speedup composites
  stroke overlaps once where the live loop composites them twice, so it would
  break the pixel-identity that history-as-paths rests on, and bitmap snapshots
  are fenced out. A depth cap or a bake-the-tail strategy is a later call, it
  interacts with undo depth, and it wants its own issue.
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
