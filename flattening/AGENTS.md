# DrawFlattening project notes

Standalone AIF command-line program that reimplements the core of the RISC
OS Draw module's path-processing pipeline from scratch: flattening Bezier
curves, dashing, thickening (stroke-to-fill), and line-end capping. It is a
learning/exercise project, not a Draw module replacement - there is no
`RMEnsure`, no SWI veneer, no CMHG. `main()` just builds a couple of test
paths in memory and runs them through the pipeline, printing and plotting
the result.

Branch `flattening-claude` off `master`; each pipeline stage was added in
its own commit (see `git log --oneline`).

## Where things are

```
h/types              Draw module data structures and constants (Draw_PathElement,
                      Draw_Tag, Draw_CapStyle, Draw_JoinStyle, Draw_LineStyle,
                      Draw_DashPattern, OS_Coord, ...)
h/bufferdata          Third-party-style buffer writer with overflow tracking (see below)
c/bufferdata
h/global              Empty placeholder required by h/bufferdata's #include "global.h"
                      (this project's per-project global header; nothing needed here yet)

h/geometry, c/geometry   Vec2 and 2D vector helpers (unit_dir, offset_point, coord<->vec)
h/pathio,   c/pathio     Reading (Subpath, path_read_subpath) and writing
                         (path_writer_init, path_write_*, path_write_result) raw
                         Draw_PathElement streams; advance_element() (see Gotchas)
h/flatten,  c/flatten    flatten() - Bezier -> LineTo flattening
h/thicken,  c/thicken    thicken() - stroke-to-fill; also holds the shared
                         offset/join engine used by cap() (see h/thicken_internal)
h/cap,      c/cap        cap() - thicken() with real line-cap styles instead of
                         butt; also holds cap_emit(), the cap-shape geometry
                         (see h/cap_internal)
h/dash,     c/dash       dash() - applies a Draw_DashPattern

h/thicken_internal      GappedPath + thicken_read_gapped_subpath() +
                         thicken_process_gapped(), exposed ONLY for c/cap to reuse -
                         not part of thicken's public API in h/thicken
h/cap_internal           cap_emit(), exposed ONLY for c/thicken to call from
                         thicken_emit_open() - not part of cap's public API in h/cap

h/fill,     c/fill       fill() - scanline rasterisation of a flattened path,
                         honouring all four Draw winding rules; calls back
                         through a caller-supplied Fill_Render (hline/polyhline)

c/main                  Test program only: show_path() (debug dump/plot),
                         report_space() (overflow reporting), render_hline()/
                         render_polyhline() (dumb Fill_Render callbacks),
                         test_flattening(), test_capping(), test_filling(),
                         main()

draw.xml                PRM-in-XML source for the Draw module chapter (data
                         structure descriptions used as the spec for this project;
                         SWIs described in it cannot actually be called - there is
                         no Draw module here, only the path-buffer format)
GOAL.md, thickening.md   Original task briefs for the flattening and thickening work
Makefile,fe1             OBJS lists every module above; TYPE=aif, LIBS=${CLIB}
```

`h/*` and `c/*` pair up by leaf name (`h/thicken` + `c/thicken`), except
`h/thicken_internal` and `h/cap_internal`, which have no matching `.c` -
they are private headers shared between exactly two translation units each.

## The pipeline

There are two pipelines here, per the PRM's own description of what
`Draw_Fill` and `Draw_Stroke` each do internally (see `draw.xml` /
`riscos-output`'s `draw.md`) - **filling a plain shape never involves caps or
joins at all**; those only exist for stroking:

```
Filling a shape:
    flatten(inpath, outpath, flatness)
        -> fill(inpath, fill_style, step, render)

Stroking a path:
    flatten(inpath, outpath, flatness)
        -> dash(inpath, outpath, pattern)          [optional - NULL pattern = passthrough]
            -> thicken(inpath, outpath, thickness) [or cap(..., style) for real line caps]
                -> fill(inpath, fill_style, step, render)
```

`thicken()`/`cap()` turn a stroke into an already-closed *fillable* outline,
which is why fill's non-zero (or even-odd) winding rule is what makes the
thickened outline read as solid rather than as a hairline outline of itself.

Each stage takes an already-processed path as `inpath` and writes a new
path into `outpath`; the test harness ping-pongs between the two static
`in_buffer`/`out_buffer` arrays with a `memcpy` between stages (see
`test_flattening()`). Every stage's `outpath` must start with a placeholder
`Draw_EndPath` element whose `data.end_path` field gives the buffer's spare
byte capacity - the standard Draw module output-buffer convention (see
`draw.xml` / the `riscos-output` skill's `draw.md`) - and every stage
returns the standard RISC OS space-report: positive spare bytes on success,
or `-(bytes that would have been required)` on overflow. See
`path_write_result()` in `h/pathio` and `report_space()` in `c/main` for how
that's produced and consumed.

`fill()` is the odd one out: it's the terminal stage, so it doesn't write a
path into an `outpath` at all - it walks `inpath` directly and calls back
through a caller-supplied `Fill_Render` (see below), the same shape as
Draw's own `Draw_Fill`/`Draw_Stroke` rendering to the VDU rather than to a
buffer. Its return value is a small error code (0 success, negative for a
specific problem), not the buffer space-report convention above.

### Key algorithms, in one line each

* **flatten()**: recursive De Casteljau subdivision of each BezierTo,
  stopping when the control-point deviation is within `flatness`.
* **dash()**: walks each subpath's total length against the dash pattern,
  writing `LineTo` for "on" runs and `GapTo` for "off" runs *within the same
  subpath* rather than starting new subpaths - see Gotchas.
* **thicken()**: offsets each subpath to both sides by half the thickness,
  joining corners with a mitre (falling back to a bevel past
  `THICKEN_MITRE_LIMIT` line-widths - same rule as `Draw_LineStyle`'s
  `mitre_limit`). A subpath with no gaps that loops back to its own start
  point becomes two closed rings wound in *opposite* directions (an
  annulus, filled correctly by a non-zero winding rule); anything else
  (open, or broken by gaps) becomes one independently-capped closed loop
  per unbroken run.
* **cap()**: identical engine to `thicken()`, but finishes each open run's
  ends with round/projecting-square/triangular geometry (`cap_emit()` in
  `c/cap`) instead of a plain butt line, driven by a real `Draw_LineStyle`.
* **fill()**: classic scanline polygon fill. For each scanline (stepped by
  `step` path units), it re-walks the whole input path, testing every edge
  for a crossing, sorts the crossings by x, then walks them left to right
  accumulating a running winding number and toggling "inside" per
  `fill_style`'s winding rule (`Draw_FillNonzero/Negative/EvenOdd/Positive`)
  to build the interior runs it hands to `render->hline`/`render->polyhline`.
  Re-walking the path per scanline (rather than building one big edge list
  up front) keeps memory use flat regardless of path complexity, at the cost
  of CPU - a deliberate frugality/speed trade-off, not an oversight. Every
  subpath is treated as implicitly closed (an edge back to its start point)
  for winding purposes, matching `Draw_Fill`'s PostScript-equivalent
  behaviour; a subpath started with `Draw_SpecialMoveTo` contributes no
  edges at all, per its definition ("doesn't affect winding numbers").

The general implementation notes worth keeping (mitre-point formula, the
two-ring closed-subpath technique, cap geometry per style, why gaps don't
break a subpath) have been written up generically in the local
`riscos-output` skill shadow's `references/draw.md`, under "Implementing
your own path thickening" - read that rather than re-deriving it, and keep
it updated if the algorithms change.

## Building, running, testing

Standard build-environment flow (see the `riscos-build-environment` skill):

```
riscos-amu
riscos-build-run aif32 --command "aif32.DrawFlattening"
```

`main()` runs `test_flattening()` (flatten -> dash -> thicken on a
curved-corner shape), `test_capping()` (all four cap styles on a bent line),
then `test_filling()` (all four winding rules on a self-intersecting
pentagram - the classic shape for telling winding rules apart: the star's
points get a winding number of 1, its self-crossed centre gets 2). The first
two print a `stage: ok (N bytes spare)` / `stage: buffer too small - ...`
line per stage via `report_space()`, then dump the resulting path as text
and plot it with `OS_Plot`. `test_filling()` instead prints one `hline
y=... x=...` line per run via the dumb `render_hline()`/`render_polyhline()`
callbacks in `c/main` (which also plot each run with `OS_Plot`), so you can
see exactly which pixels each winding rule decided were interior.

**To see the plotted graphics without the console text dump obscuring
them**, redirect the program's stdout when running it:

```
riscos-build-run aif32 --command "aif32.DrawFlattening > stdout" \
    --command "*screensave -native screen" --return-file screen --return-to screen.png
```

Then view `screen.png` (it's actually a Sprite despite the name). Without
the `> stdout` redirect, the text dump fills most of the screen and hides
the plot.

## Caveats and gotchas found while building this

* **Never assume a fixed path-element word size.** Elements are variable
  length (Close = 1 word, MoveTo/LineTo/GapTo = 3, BezierTo = 7,
  Continuation/EndPath = 2) - always dispatch on the tag via
  `advance_element()` (`h/pathio`). An earlier version of `flatten()` used a
  fixed "+2 words" fallback for tags it didn't explicitly handle, which
  silently corrupted `CloseGap`/`CloseLine` elements (1 word) and dropped
  them from the flattened output - closed subpaths never reached
  `thicken()` correctly until that was fixed.
* **Silent buffer overflow is a real trap.** Before the `bufferdata`
  refactor, hand-rolled space checks meant that once a buffer got low, a
  small write (eg a 1-word Close) could still succeed even after a bigger
  write (a 3-word MoveTo/LineTo) had already silently failed - producing
  dangling `CloseLine` tags with no geometry before them, and quietly
  losing the rest of the path with no error. This is exactly why every
  stage now goes through `bufferdata_t` and reports back
  spare-space-or-required-space rather than just writing-and-hoping. If you
  add a new writer anywhere in this pipeline, route it through
  `path_write_*` (`h/pathio`), not raw pointer arithmetic.
* **A `Draw_CloseGap`/`GapTo` edge is unstroked but does not split the
  subpath.** It's tempting to treat a gap as "end this subpath, start a new
  one", but Draw's own definition is "do not start a new subpath" - one
  subpath can contain many gaps and still be one Move/Close unit. Get this
  wrong and dashed output silently strokes solid across every gap (this
  bug existed briefly: `thicken()` initially lumped `GapTo` in with
  `LineTo` when reading points, before `GappedPath`/`thicken_internal.h`
  were introduced to track gap-before-vertex per point).
* **Round caps/joins here are polygon approximations (8 segments), not true
  arcs.** Real Draw emits an actual curve and re-flattens it (fill style
  bit 30). This project stays flatten-once and just fans out straight
  segments directly - fine visually, but not bit-compatible with Draw's own
  output if that ever matters.
* **A triangular cap's width must exceed 0.5 line-widths (256th-units
  `width_256 > 128`) to look like an arrowhead rather than a tapered
  point** - at exactly 128 the base sits flush with the ordinary stroke
  edges and it just looks like the line coming to a point. `test_capping()`
  uses 384 (1.5x) to make this obvious.
* **Test buffers need real headroom.** `in_buffer`/`out_buffer` are 1024
  words (4KB); 256 words (1KB) is enough for the individual cap-style
  tests but not for the combined flatten+dash+thicken pipeline test on a
  curve - a dashed, thickened curve needs far more path elements than the
  same curve undashed. `report_space()` will now tell you exactly how much
  more is needed rather than silently truncating.
* **Filling a shape does not involve caps at all - only stroking does.**
  It's tempting to assume every pipeline ends in cap()/thicken() before
  rendering, but `Draw_Fill`'s own documented processing order is just
  "close open subpaths -> flatten -> transform -> fill"; caps and joins are
  purely a stroking concept (`Draw_Stroke`'s "thicken path / add caps and
  joins" step). Only feed a path through `thicken()`/`cap()` before `fill()`
  when you are actually rendering a *stroke* - a plain filled shape goes
  straight from `flatten()` to `fill()`.
* **`fill()`'s `step` parameter is a scanline granularity in path units, not
  a pixel count.** The path coordinates here are unscaled Draw-ish integers
  (the same ones `show_path()`'s `X()`/`Y()` macros divide by 64 for
  display), so scanning every single unit would be enormously wasteful;
  `test_filling()` uses `step = 64` to land one scanline per displayed
  pixel row. Pick `step` to match whatever the render callbacks actually
  plot at, or interior detail finer than one step can be missed entirely.
* **A `Draw_SpecialMoveTo` (type 3) subpath is invisible to winding.** Per
  its PRM definition it "does not affect winding numbers when filling" -
  `fill()` deliberately skips generating any edges for such a subpath
  rather than treating it like an ordinary `Draw_MoveTo`. Get this backwards
  and a special-move subpath meant as an inert construction aid ends up
  cutting a hole (or adding unwanted coverage) in the fill.
* **Winding sign is a coordinate-order accident, not something to hand-
  predict.** `test_filling()`'s pentagram happens to wind negative rather
  than positive under this project's `y increases downward` edge-crossing
  convention (`fill_test_edge()`'s `winding = b.y > a.y ? 1 : -1`) - so in
  that test, `Draw_FillNegative` fills solid and `Draw_FillPositive` fills
  almost nothing, the opposite of what a quick shoelace-formula guess by
  eye would suggest. Verify winding sign empirically (or trace it through
  the actual vertex order) rather than assuming a "conventional" CCW/CW
  direction is positive.
