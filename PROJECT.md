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

**Increment 3 (day 009 revisit — the first thing you see is the toy drawing).**
The open thread "First load shows nothing the toy makes" is the biggest gap
between what the toy is and what a stranger meets: the demo URL opens on a
black rectangle, a bar of mostly-disabled controls, and one line of text asking
for a gesture before anything has shown what a gesture buys. The README's
screenshot does that job today, which works from the README and not at all from
the URL. This increment closes it, and closes three named defects from the
day-006 evening pass that were recorded rather than fixed.

That thread asked for three decisions before any code. The planner settles them
here:

- **Is the opening flourish undoable?** No. It is not a history entry. Undo and
  Redo stay exactly as disabled on first load as they are today.
- **Does it count as ink for Clear and Save PNG?** No. `hasInk()` is untouched,
  so Clear and Save PNG stay disabled while it is on screen and it can never
  reach an exported PNG. This is what makes the feature safe: on first load
  every control that could observe it is already disabled for other reasons.
- **How does it get out of the way?** The first `pointerdown` on the canvas
  erases it within one frame, and it never returns for the life of the page.
  A control dismisses it too, but only a control that actually changes
  something: picking a *different* colour or a *different* pen. Pressing the
  swatch or the pen that is already active changes nothing, so it dismisses
  nothing — measured, the flourish is still at full ink afterwards. The other
  four controls are all disabled on an empty canvas, so no path runs from them
  to a live flourish. The ghost is a first-run state, not a mode.

In scope:

- **The opening flourish.** With history empty and no input yet received, the
  page draws one stroke of its own: a scripted pointer path fed through the
  same pen physics the toy uses, painted at reduced opacity so it reads as a
  demonstration rather than as the user's drawing. It animates in over roughly
  2.4 s and then rests until dismissed. The scripted path is deterministic and
  sized from the canvas, so it composes with the existing pen scale rather than
  fighting it.
- **Reduced motion is respected.** Under `prefers-reduced-motion: reduce` the
  flourish is present in full on load with no animation — a stranger who has
  asked the OS for stillness still sees what the toy makes.
- **`Saved ✓` cannot land on a disabled button** (open thread, day-006 evening
  fourth pass). `ackSave()` runs in the async `toBlob` callback while the guard
  that drops it runs synchronously, so emptying the canvas during the blob
  encode leaves a dimmed `Saved ✓` for the rest of the 1.6 s timer. The fix is
  the one the thread names: re-ask the ink question inside the callback.
- **The off-canvas hint names a remedy the device has** (open thread, "two
  chrome nits"). `off-canvas — widen the window` is reachable on a phone, where
  widening is not an affordance the device offers.
- **A control that disables itself keeps focus somewhere real** (same thread).
  Today Undo, Redo, Clear and Save PNG drop focus to `<body>` when they go
  disabled under the keyboard, so the next Tab restarts at the first swatch.

**Increment-3 fence (added to the standing fence above).** The flourish is not
persisted, not undoable, not exported, not replayable, has no controls, does
not loop, and does not return. The standing fence's "no animation" line is
opened *only* for this one first-run draw-in — no other motion, no timeline, no
replay UI. No new file, no dependency, no build step. Nothing in this increment
changes stroke physics, the palette, the pens, or what `redraw()` paints from
history: an existing drawing must render pixel-identically to the day-006
build.

**Done-checklist (increment 3).**

1. First load at 1440x900 and at 375x667 shows ink from the opening flourish
   within 3 s with no pointer input whatsoever.
2. While the flourish is on screen, Undo, Redo, Clear and Save PNG are all
   still disabled — the flourish is not ink.
3. The first `pointerdown` erases it within one frame, and it never reappears:
   not after a stroke, an undo, a clear, a resize, or a dpr change.
4. Under `prefers-reduced-motion: reduce` the flourish is complete on load and
   nothing animates.
5. `save.click(); clear.click();` in one task leaves no `Saved ✓` on a disabled
   button.
6. At 375x667 the off-canvas hint names a remedy the device actually has.
7. Tab to Undo, press it until it disables, and focus is still on a real
   control rather than `<body>`. Same for Redo, Clear, Save PNG.
8. Still one HTML file and exactly one network request; no console errors or
   warnings; no page scroll at 320 px; a drawing made before this increment
   renders pixel-identically.

**Rubric lines that matter most here:** delight (this is the increment whose
whole purpose is the first ten seconds), README truthfulness (the README must
describe the flourish exactly as it behaves, including that it is not part of
the drawing), and scope discipline (the fence above is narrow on purpose — the
temptation is a looping demo reel).

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
      empties inside the 1.6 s window; that half is **not** closed as of day
      006, and the line here claiming it was has been corrected. A fourth
      independent verifier reproduced the dimmed `Saved ✓`; it was carried as
      an open thread until increment 3 closed it (see day 009 below).
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

**Increment 3 (day 009)** — the opening flourish, plus the three day-006
residuals that were recorded rather than fixed. The feature shipped and then
went through a fix cycle the same day: three independent critics raised ten
defects against it, six of them against the flourish itself. The lines below
say what stands after that cycle, not what the first pass wrote; every one was
checked by driving the build, and the compositing lines were checked by looking
at the pixels, because the first pass's bug was invisible in the diff and
obvious in a screenshot.

- [x] The opening flourish. With history empty and no input received, a
      scripted figure-eight pointer path is fed through the real `orbit`
      physics — one fixed 60fps step per sample, so `frames === 1` and the lerp
      is `pen.chase` itself — and drawn in over 144 frames (2.4 s), then rests.
      Not a history entry, not undoable, not ink for `hasInk()`, never in an
      export. It is repainted from `redraw()` *after* the history it stands in
      for and recorded nowhere, so a resize or a dpr change rebuilds it against
      the new canvas at the same point in its draw-in. Its amplitudes are
      measured in orbit radii and then clamped to the canvas, so the
      composition is the same drawing at every size.
- [x] It composites **once**. The first pass stroked all 144 segments onto the
      canvas under `globalAlpha = 0.42`, so every round cap overprinted its
      predecessor and the alpha stacked as `1-(1-0.42)^n`: max luminance down
      the thick left loop alternated 112 / 168 on a ~3.5 css-px period at
      1200x800/dpr 2, and at 375x667 the caps stacked 4–5 deep and the median
      ink read 168. A string of pearls, from the one feature whose whole job is
      to show what the toy's line looks like. The segments now go into an
      offscreen layer at alpha 1 — where overlaps resolve exactly as they do in
      a real stroke — and the layer is blitted once. Measured after: every
      inked pixel reads R=113, `min = p50 = p90 = max`, at 1440x900, 1200x800,
      1024x768, 375x667 and 320x568 alike.
- [x] And it is fainter than the page's instruction at every size. Re-measured
      this pass, because the figure recorded here first (5.76:1 for the hint)
      was asserted rather than measured and is not a number this CSS can
      produce — it is what alpha 0.56 would give, and the hint is at 0.55.
      Both sides were taken off the built page. The flourish: the modal inked
      pixel is rgb(113,110,106) at 1440x900 dpr 1 and dpr 2, 375x667 dpr 2 and
      320x568 dpr 2 alike, which is **3.72:1** against `#111` (the 3.74 here
      before came of treating that pixel as a neutral grey 113). The hint: a
      clipped screenshot of the text box decoded back through a canvas gives a
      fully-covered glyph of rgb(142,138,133) at dpr 1 and dpr 2 alike, which
      is **5.51:1**; computing the same colour from the CSS instead —
      `rgba(245,239,230,0.55)` over `#111` — gives 142.40,139.10,134.15 and
      **5.58:1**. The two disagree by one 8-bit step on green and blue, which
      is Chromium's compositor truncating where the arithmetic rounds; 5.51 is
      what is on the screen and 5.58 is what the stylesheet asks for. Either
      way the conclusion is the same one and it is not close: the
      demonstration sits below the instruction in the visual hierarchy at every
      size, where before it beat it 7.7:1 to 5.7:1 at phone width and the
      README's word "faint" was measurably false there.
- [x] It is routed clear of the hint. Both were centred in `#stage`, so the
      figure walked through the page's only instruction: 11.3% of the hint's
      text box was flourish ink at 375x667, 6.3% of it at R≥150, and a glyph
      over a bead measured 1.07:1. `buildGhost` now measures the hint's text
      box — a Range over its contents, so a two-line hint at 320px is measured
      as two lines — takes the taller strip that leaves, and centres the figure
      in it, reducing the script's vertical amplitude if the figure will not
      fit. The orbit radius is never touched: shrinking that would misrepresent
      the pen. Measured share of the hint's text box that is flourish ink,
      after: 0.00% at 1440x900, 1200x800, 1024x768, 667x375, 600x300, 375x667
      and 320x568.
- [x] No cusp in the curve. A hard notch sat on the left of the figure at every
      size — half a line width of jog, reading as a rendering glitch. It was
      the pen stalling: drawn pen speed is the centre's speed plus the orbit's
      own tangential speed (`radius * angVel`, 8.80 px/frame for `orbit`), and
      where those are equal and opposing the pen reverses. Pen speed fell to
      0.96 px/frame at 1440x900 and 0.08 px/frame at 375x667. The pen's
      constants cannot move; the starting orbit angle can, and it decides which
      side of its orbit the pen is on when the centre coasts through that
      speed. Swept over the whole circle at seven canvas sizes: 0 is the worst
      value there is, and there is a plateau from ~160° to ~225°. π is the
      middle of it and the one value with a reason — the pen starts on the far
      side of its orbit, broadside to the turn. After: minimum drawn pen speed
      3.05 px/frame at 1440x900 (0.35 of the orbital speed) and 2.75 px at
      375x667 (0.53), sharpest turn between adjacent samples 39–42° rather than
      82–132°.
- [x] Reduced motion, on load *and* at runtime. Under
      `prefers-reduced-motion: reduce` the flourish is complete before the
      first paint and nothing animates. The query was sampled once at init and
      never watched, so flipping it 30 frames into the draw-in changed nothing
      (ink kept climbing 2344 → 3238 → 3929 over the next 20 frames); it is now
      watched, and a change to `reduce` *finishes* the flourish rather than
      freezing it half-drawn — 9819 inked px 500 ms in, 36604 within one frame
      of the toggle, which is exactly what a page loaded with `reduce` renders.
- [x] Only input that does something dismisses it. The first pass dismissed on
      any `pointerdown` on the canvas (so a right-click did), on any
      `undo()`/`redo()` (so Ctrl+Z against an empty history did), and on any
      button in the bar (so pressing the already-selected colour or pen did).
      The two history actions now return early when there is nothing to do, the
      canvas dismisses after the primary-button test, and the bar's blanket
      listener is gone — the swatch and pen handlers dismiss on the branch that
      changes the selection. Measured at 1200x800/dpr 2, inked px unchanged at
      36604 for right-click, middle-click, Ctrl+Z / Ctrl+Y / Ctrl+Shift+Z on an
      empty history, a click on the bar's padding, and the already-active
      swatch and pen by pointer and by Enter; 36604 → 0 for a different swatch
      or a different pen. A primary press still erases it in the same task as
      the `pointerdown`, before the next frame.
- [x] The canvas is in the accessibility tree. It had no `aria-label`, no role
      and no fallback content, so nothing told a screen-reader user the page had
      drawn anything. It is now `role="img"`, described by the hint, and named
      from `updateChrome()` — the one place that already knows the state — so
      the name cannot outlive what it describes: Chrome reports "one faint
      stroke the toy drew itself…" while it is up, then "1 stroke", "2
      strokes", "1 stroke" after Undo, "empty" after undoing to nothing and
      after Clear.
- [x] `Saved ✓` cannot land on a disabled button (day-006 residual). The ink
      question is asked again inside the `toBlob` callback, on the far side of
      the encode. `save.click(); clear.click();` in one task leaves the label
      reading `Save PNG` on the disabled button.
- [x] The off-canvas hint names a remedy the device has (day-006 residual). On
      a touch device the copy is `off-canvas — rotate to bring it back`;
      re-driven by drawing at the right edge of a 667x375 landscape phone and
      rotating it to 375x667.
- [x] A control that disables itself hands focus on (day-006 residual). Undo
      pressed until it disables leaves focus on Redo; Clear pressed until it
      disables leaves focus on Undo. Neither drops to `<body>`.
- [x] README made true: the flourish described exactly as it behaves, including
      that it is not part of the drawing and what does and does not dismiss it,
      and both prose sections brought inside STYLE.md's sentence limits.
- [x] `screenshot.png` recaptured from this increment's build, at `3ee0181`
      (verified: 2400x1600 in the file, which is 1200x800 at
      deviceScaleFactor 2). Three strokes, one per pen, each carried through
      its settle so the closing curl the pen leaves is in frame: `drift` in
      gold, `orbit` in warm white, `coil` in sky blue. It is a shot of the toy
      being *used*, so the flourish has been dismissed and is deliberately not
      in it — the increment's headline feature is the thing this image cannot
      show, and the README's opening sentence is what carries it instead. The
      alt text describes the drawing and the bar; there is no caption paragraph,
      because STYLE.md's template has no slot for one. Today's flourish fixes
      do not stale it: the flourish is not in the frame, and stroke rendering
      is pixel-identical to `c84b362` (re-proved below).
- [x] Still one HTML file and exactly one network request at 1440x900, 375x667
      and 320x568; console clean on every run above; no page scroll at 320 px
      (scrollWidth 320 = clientWidth, scrollHeight 568 = clientHeight).
- [x] Pixel identity re-proved after the compositing change, not assumed. A
      harness installs a fake `requestAnimationFrame` before the page script
      runs, so both builds integrate the same `dt` sequence, drives one scripted
      90-sample gesture through real pointer events, and compares
      `canvas.toDataURL()` against `c84b362`. 36 comparisons identical: 8
      viewport/dpr combinations (1440x900 at dpr 1 and 2, 1200x800 dpr 2,
      1024x768 dpr 1.5, 375x667 at dpr 2 and 3, 320x568 dpr 2, 667x375 dpr 2,
      600x300 dpr 1) across all three pens and five colours, each run three
      ways — from a cold load, with the flourish live at `pointerdown`, and
      with an undo+redo replay after the stroke.

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

- **What first load demonstrates is one pen, and only one.** The old thread
  ("First load shows nothing the toy makes") is closed: increment 3 shipped the
  opening flourish, and after the fix cycle it composites once, reads 3.74:1
  against the background at every size, clears the hint's text box entirely,
  and has no cusp left in it. What is narrower and still open is that it is
  drawn with `orbit` alone. `coil` and `drift` sit named in the bar with
  nothing on screen to say what either draws, and the phone thread above says
  those two pens are the ones that compress into each other at 375 px — so the
  size where the difference is hardest to see is also the size where nothing
  demonstrates it. Not fixed here, and not a fix-cycle call: three flourishes
  would be a demo reel, which the increment-3 fence excludes by name, and one
  flourish per pen picked on first press is a different feature with its own
  question (does picking a pen you have not used replay it?). It wants an owner
  issue.

- **The flourish's compositing layer is the one bitmap in the build.** Drawing
  it once at one alpha means an offscreen canvas — cut to the figure's bounding
  box rather than the canvas, so 1117x486 and 2.1 MB at 1440x900/dpr 2 against
  the 18.5 MB the 2880x1686 backing store would cost, and 0.5 MB against 2.9 MB
  at 375x667/dpr 2 — released the moment the flourish is dismissed.
  It is a transient compositing buffer for a view-only overlay, not a bitmap in
  the history path, and the "history holds no bitmaps" rule above is unchanged.
  It is recorded here because it is the only allocation of its kind in the file
  and the next person to read the memory notes deserves to find it. The
  alternative that avoids it — one `Path2D` per constant-width run — does not
  work for this path: the width changes on almost every one of the 144 samples,
  so the runs are one segment long and the overlaps composite twice again.

- **Two things about the flourish are tuned against a list of sizes, not
  proved.** The start angle that removes the cusp (π) was chosen by sweeping
  the whole circle at seven canvas sizes and taking the middle of the plateau;
  the vertical-amplitude shrink that keeps the figure inside the strip beside
  the hint converges by iteration, bounded at eight passes. Both were checked
  at 1440x900, 1200x800, 1024x768, 667x375, 600x300, 375x667 and 320x568, and
  the shrink has a floor that degrades to a flat run with the pen's loops still
  on it rather than to something off-canvas. Neither is a proof for every
  canvas size a browser can produce. The cheap guard if this ever bites — a
  post-build assertion that the path clears the hint box and the edges, falling
  back to no flourish — is a fix, not a feature, but nothing has been seen to
  need it yet.

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
