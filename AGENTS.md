# Draw module notes

This is a from-scratch reimplementation of the RISC OS Draw module,
including its path-processing pipeline: flattening Bezier curves, dashing,
thickening (stroke-to-fill), line-end capping, and scanline filling with all
four Draw winding rules. The pipeline logic (`c/flatten`, `c/dash`,
`c/thicken`, `c/cap`, `c/fill`, plus their shared helpers) started life as a
standalone AIF test program under `flattening/` and has since been folded
into the module's own `c/`/`h/` directories so `c/module`'s SWI veneers can
call straight into it. `flattening/` now only holds reference material
(`draw.xml`, `GOAL.md`, `thickening.md`) and old build output - the working
source lives at the top level from here on.

Branch `flattening-claude` off `master`; each pipeline stage was added in
its own commit (see `git log --oneline`).

## Where things are

```
h/modhead                CMunge-generated module header (SWI numbers, error
                          blocks, Mod_Init/Mod_Final/SWI_* prototypes) - do
                          not hand-edit, it's regenerated from cmhg/modhead
cmhg/modhead              CMHG source: SWI chunk, error-identifiers, the
                          swi-decoding-table - this is what h/modhead comes from
c/module                 SWI_* veneers - decode _kernel_swi_regs into typed
                          locals, call the pipeline functions below, encode
                          the result back into regs

h/types                  Draw module data structures and constants (Draw_PathElement,
                          Draw_Tag, Draw_CapStyle, Draw_JoinStyle, Draw_LineStyle,
                          Draw_DashPattern, OS_Coord, Draw_FillStyle, ...)
h/bufferdata, c/bufferdata   Third-party-style buffer writer with overflow
                          tracking (see below)
h/global                 Empty placeholder required by h/bufferdata's
                          #include "global.h" (the project's per-project
                          global header; nothing needed here yet)

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
h/fill,     c/fill       fill() - scanline rasterisation of a flattened path,
                         honouring all four Draw winding rules; calls back
                         through a caller-supplied Fill_Render (hline/polyhline)
h/transform, c/transform transform() - applies an OS_Trfm matrix to every
                         coordinate in a path (including BezierTo control
                         points - the one stage that doesn't need a
                         flattened input first)

h/thicken_internal      GappedPath + thicken_read_gapped_subpath() +
                         thicken_process_gapped(), exposed ONLY for c/cap to reuse -
                         not part of thicken's public API in h/thicken
h/cap_internal           cap_emit(), exposed ONLY for c/thicken to call from
                         thicken_emit_open() - not part of cap's public API in h/cap

c/test                  Test program only (built as a standalone AIF, not
                         part of the module): show_path() (debug dump/plot),
                         report_space() (overflow reporting), render_hline()/
                         render_polyhline() (dumb Fill_Render callbacks),
                         test_flattening(), test_capping(), test_filling(),
                         main()

Makefile,fe1             The module's own build (CModule) - builds c/module
                         plus every pipeline .c file into oz32/, then rm32
MakefileTest,fec          Secondary makefile (LibraryCommand) for the c/test
                         AIF - builds the same pipeline .c files again into
                         o32/ (a different object directory, so it never
                         clashes with the module build) plus o.test, linking
                         to aif32.DrawTest. Not picked up automatically by
                         plain `riscos-amu` - invoke it explicitly (see
                         Building, running, testing below)

flattening/draw.xml               PRM-in-XML source for the Draw module chapter
flattening/GOAL.md, thickening.md  Original task briefs for the flattening and
                                   thickening work
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
`test_flattening()` in `c/test`). Every stage's `outpath` must start with a
placeholder `Draw_EndPath` element whose `data.end_path` field gives the
buffer's spare byte capacity - the standard Draw module output-buffer
convention (see `draw.xml` / the `riscos-output` skill's `draw.md`) - and
every stage returns the standard RISC OS space-report: positive spare bytes
on success, or `-(bytes that would have been required)` on overflow. See
`path_write_result()` in `h/pathio` and `report_space()` in `c/test` for how
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
break a subpath, the winding-rule/scanline algorithm) have been written up
generically in the local `riscos-output` skill shadow's `references/draw.md`
- read that rather than re-deriving it, and keep it updated if the
algorithms change.

## Module integration status

The module is wired up and working - `c/module`'s SWI veneers call straight
into the pipeline, and this has been verified against the actual built
module (`rm32/Draw`), not just `c/test`: `RMLoad`ed and driven via BASIC
`SYS` calls, `Draw_Fill` and `Draw_Stroke` both render correctly on screen,
and `Draw_FlattenPath`/`Draw_TransformPath`/`Draw_StrokePath`/
`Draw_ProcessPath` were checked numerically (exact byte counts for a known
input, an exactly-translated coordinate through a real matrix, and a real
result agreeing with its own size-query mode).

* **Floating point.** The pipeline originally used `double` (sqrt/sin/cos)
  throughout - fine for a standalone AIF, but modules shouldn't use FP
  without wrapping SVC-mode entry with `Asm/fpsvc.h` save/restore (see
  `using-libasm`). It's since been reworked to integer/16.16 fixed-point
  throughout (`Vec2`/`Dir2` in `h/geometry`) so the module never touches the
  FPU. This compiler has no native 64-bit integer type at all (not even via
  `stdint.h`), which shaped the approach: `muldiv()` (`Asm/muldiv.h`, from
  libAsm - link `${ASMLIB}`/`${ASMLIB_ZM}`) handles every "multiply two
  32-bit values, then divide by a third"; a sum of two large products (eg a
  cross product) is restructured so each product is divided back down to a
  normal scale *before* combining; and the one genuine 64-bit need (isqrt of
  a sum of two squares, for vector/mitre/dash-segment length) uses a small
  self-contained 64-bit emulation in `c/geometry` (16-bit partial-product
  multiply, carry-aware add/sub) rather than pulling in the environment's
  own LongLong library for one function. See `c/geometry`'s file comment for
  the full reasoning, and the "Cap and Join Specification" section of
  `draw.xml` for why 16.16 is the natural scale (it's what
  `Draw_LineStyle.mitre_limit` already uses).
* **SWI wiring.** `Draw_ProcessPath` (buffer-output modes only),
  `Draw_Fill`, `Draw_Stroke`, `Draw_StrokePath`, `Draw_FlattenPath` and
  `Draw_TransformPath` are implemented. Multi-stage SWIs share
  `run_pipeline()` (`c/module`), which `malloc()`s a fresh scratch buffer
  per stage and frees the previous one as soon as it's consumed - SVC-mode
  module code can't safely use large stack arrays the way `c/test` used
  static globals, so each intermediate buffer is heap-allocated instead,
  sized generously relative to its input (`scratch_alloc()`) rather than
  predicted exactly. `deliver_path()` then either copies the final result
  into the caller's buffer or reports its required size, and reports
  `Error_DrawPathFull` on any allocation failure or overflow anywhere in the
  chain. `Draw_Fill`/`Draw_Stroke` render to the VDU via a small
  `Fill_Render` backed by `OS_Plot` (`module_hline()`/`module_polyhline()`
  in `c/module`) - this is also where Draw's internal units (`Draw_OSUnit` =
  256 per OS unit) get converted down to the OS units `OS_Plot` expects,
  since every pipeline stage (including `transform()`) leaves coordinates in
  internal units throughout.
* **Known gaps, left out of scope for this pass:**
  - `Draw_FillClipped`/`Draw_StrokeClipped` (RISC OS 5+, clip descriptors)
    and the FP-vector SWI slots (`Draw_1`, `Draw_3`, ... - the unnamed
    odd-numbered entries in the swi-decoding-table, auto-handled by CMHG)
    report `err_DrawUnimplementedDraw`/`error_BAD_SWI` respectively.
  - `Draw_ProcessPath`'s R7=1/2 (VDU output) are unsupported by design, per
    the PRM itself ("This call is also unable to handle R7 = 1 or 2");
    R7=&80000000+ptr (bounding box output) and R7=0 (write back in place)
    are gaps, not PRM-documented exclusions - both report
    `err_DrawUnimplementedDraw`.
  - "Close open subpaths" (fill style bit 27) isn't implemented as a
    distinct pass. `fill()` already closes every subpath implicitly for
    winding purposes, which covers the common case, but nothing physically
    inserts a closing element for other output modes.
  - The "DrawV" vector-claiming mechanism (`DrawV_Reason` in `h/types`, for
    printer drivers etc. to intercept Draw calls) is entirely out of scope.
  - Fill style bits 2-5 (boundary-pixel control) aren't honoured by
    `fill()` - only the winding-rule bits (0-1) are. This also means the
    documented difference between fill style `&30` and `&18` (Draw_Stroke's
    default when R4=0, meant to select boundary-only plotting for a
    zero-width path) has no effect here; hairline stroke width instead
    falls back to `DRAW_HAIRLINE_THICKNESS`, a small nonzero thickness, so
    something visible is still produced.
  - `Draw_TransformPath`'s R1=0 buffer (and every other stage's scratch
    buffer) is sized by a fixed multiplier of the input's byte size
    (`SCRATCH_MULTIPLIER`/`SCRATCH_MINIMUM` in `c/module`), not calculated
    exactly - pathologically point-dense paths could in principle still
    overflow it and report `Error_DrawPathFull` where a real Draw module
    would succeed.

## Building, running, testing

Standard build-environment flow (see the `riscos-build-environment` skill).
Two separate makefiles live at the top level:

```
riscos-amu                                     # builds the Draw module itself (Makefile,fe1 / CModule)
riscos-amu -f MakefileTest,fec                 # builds the standalone test AIF (aif32.DrawTest)
riscos-build-run aif32 --command "aif32.DrawTest"
```

`main()` in `c/test` runs `test_flattening()` (flatten -> dash -> thicken on
a curved-corner shape), `test_capping()` (all four cap styles on a bent
line), then `test_filling()` (all four winding rules on a self-intersecting
pentagram - the classic shape for telling winding rules apart: the star's
points get a winding number of 1, its self-crossed centre gets 2). The first
two print a `stage: ok (N bytes spare)` / `stage: buffer too small - ...`
line per stage via `report_space()`, then dump the resulting path as text
and plot it with `OS_Plot`. `test_filling()` instead prints one `hline
y=... x=...` line per run via the dumb `render_hline()`/`render_polyhline()`
callbacks in `c/test` (which also plot each run with `OS_Plot`), so you can
see exactly which pixels each winding rule decided were interior.

**To see the plotted graphics without the console text dump obscuring
them**, redirect the program's stdout when running it:

```
riscos-build-run aif32 --command "aif32.DrawTest > stdout" \
    --command "*screensave -native screen" --return-file screen --return-to screen.png
```

Then view `screen.png` (it's actually a Sprite despite the name). Without
the `> stdout` redirect, the text dump fills most of the screen and hides
the plot.

### BASIC screenshot comparison suite (`tests/basic/`)

Nine BASIC programs drive the module's SWI veneers directly (rather than
calling the pipeline C functions like `c/test` does) and `*ScreenSave` the
result, so the same script can be run unchanged against either the
project's own built module or a real reference `Draw` module for a visual
diff:

```
TestFill          Draw_Fill, all four winding rules, on the same pentagram as c/test's test_filling()
TestCurve         Draw_Fill on a path with a BezierTo edge, at two flatness values
TestStroke        Draw_Stroke, all four cap styles, on the same bent line as c/test's test_capping()
TestDash          Draw_Stroke with a dash pattern, on an open line and a closed square
TestTransform     Draw_Fill with a non-identity trfm (rotation + scale + translation)
TestMultiSubpath  Draw_Fill on a path with two independent subpaths in one call (disjoint squares, and an evenodd hole)
TestSpecialMove   Draw_Fill where a second subpath starts with Draw_SpecialMoveTo, which must not affect winding
TestClosedStroke  Draw_Stroke on a closed subpath (the two-ring annulus case, distinct from an open/capped run)
TestDashMulti     Draw_Stroke with a 4-element dash pattern and a nonzero pattern "start" phase offset
```

All nine currently match a real reference module pixel-for-pixel (bar a
handful of sub-pixel-rounding pixels along dash boundaries in
`TestDashMulti`, not a structural mismatch).

Run one against the built module:

```
riscos-build-run rm32/Draw,ffa tests/basic/TestFill,fd1 \
    --command "RMLoad Draw" --command "Run TestFill" \
    --return-file screen --return-to screen.png
```

A real reference `Draw` module can be exercised the same way via
`riscos-run` (needs a local Pyromaniac runtime - see the
`debugging-with-pyromaniac` skill's `references/zeropage.md` for the
`--config memorymap.zeropage_enable=yes`/`--config
watchregions.lowmemory=no` incantations a genuine binary module typically
needs, and for the fact that it may depend on other resident modules (eg
`OSSWIs`) that aren't part of this project).

**When comparing against a real Draw module, always pass real values, not
`0`/NULL placeholders, for `Draw_Stroke`'s `trfm`, `flatness`, and
`line_style`, and always give `line_style.mitre_limit` a real nonzero value
(eg `&40000` = 4.0 in 16.16) whenever the path being stroked has an actual
corner.** This project's own `Draw_Stroke` veneer tolerates `0`/NULL there
and substitutes sensible defaults (see "Defaults substituted..." in
`c/module`), but a real Draw module does not: it silently renders nothing
at all (no error) if these are left as `0`, which looks identical to "the
stroke pipeline is broken" until you compare against a minimal known-good
call and narrow down which argument is at fault. `Draw_Fill` does not have
this problem - `0`/NULL there behaves as documented on both this project's
module and a real one.

**A real Draw module also appears to drop an entire dashed stroke below
about 2 OS units of thickness** (512 in Draw units, `Draw_OSUnit*2`) -
found via `TestDashMulti`, whose original 500-unit thickness rendered on
this project's module but nothing at all on a real one; 512+ renders in
both. This project's `dash()`/`thicken()` have no such lower bound.
Whether this is a genuine minimum-width rendering rule in real Draw or
specific to the reference binary tested isn't confirmed - noted here so a
future thin-dashed-line comparison isn't mistaken for a broken test.

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
* **Modules can't casually use floating point.** `writing-cmodules`
  guidance is explicit that FP needs `Asm/fpsvc.h` save/restore around SVC
  mode entry, or should just be avoided. Since this pipeline started as an
  AIF-only exercise, it leant on `double` (sqrt for vector length/mitre
  length, sin/cos for round caps) throughout - all of that has to be (or is
  being) converted to fixed-point/integer arithmetic before `c/module` can
  safely call into it. `c/test`'s AIF build is unaffected either way (user
  code doesn't have the same restriction), so this is purely a
  module-integration concern, not a test-harness one.
