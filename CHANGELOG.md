# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.7.0] - 2026-09-15 — dashes, focal gradients, real alpha, and a cap that stops being a cliff

Six items over two waves. ⛔ **Everything a 0.6.0 caller does is byte-identical for every input that
worked in 0.6.0**: agnos's `tests/gpu/refagree.cyr` prints **BYTE-IDENTICAL on all 200 paths** against
this `dist/`, rekha (23 suites) and dhancha (18) pass against it, and the 19 pre-0.7.0 suites pass with
their assertions unedited apart from `area_test` group N, whose 4 assertions pin a limitation this
release deliberately removes.

### Fixed — `SD_FLATTEN_CAP` stops being a cliff: every store that DROPPED data now grows

Through 0.6.0 a path over 8,192 edges filled WRONG (contours left open), a stroke run stopped at point
8,192, `sd_path_flatten` returned 8,192 points, and a styled stroke longer than one batch met itself by
MAX where it crossed. The fill's edge list, the crossings, the stroker's run, the curve flags and the
styled batch now DOUBLE when full — from the GLOBAL `alloc`, never the consumer's hook (0.5.5's rule:
a hook is an arena that gets reset) — and `sd_path_flatten`'s result doubles through `sd_alloc`,
because it is the caller's result on the seam.
⭐ MEASURED, a 20,000-gon disc of radius 120 px on 256x256: 0.6.0 filled **24.5 %** of its ink and left
the centre pixel 0; 0.7.0 fills **99.997 %** of 255·πr² (AREA 99.995 %).
⛔ INITIAL CAPACITIES ARE UNCHANGED, so every first-use figure a consumer has measured is still exact
(fill 590,152 B, styled batch 270,592 B, an AREA styled stroke's first-ever coverage work 860,688 B),
and growth is monotone: a process pays for its largest path once, then **0 B** a call.
⚠ **One styled stroke is now ONE batch** (`SD_STROKE_BATCH_CAP` defaults to 0 = no limit), which
removes the cross-batch MAX loss: MEASURED, 60 random self-crossing walks of 700–1,100 segments —
0.6.0 split 20 (worst 118 levels), 0.7.0 is byte-identical on all 60, and on 60 more of 2,000–5,000
segments where 0.6.0 split every one (worst 131). The flush + re-emission machinery is KEPT as the
fallback for a set cap and for a failed growth. One batch is 5–8 % slower on a big self-crossing
stroke (12,700-edge Lissajous: 41.3 ms vs 39.2 ms) — flushing is faster precisely because each flush
sweeps only its own rows, which is what bought the loss.
⚠ The `SD_AA_AREA` dispatch now falls back on TRUNCATION, not on size, so an AREA fill of exactly
8,192 edges runs the area engine (MEASURED 41 ms, where 0.6.0's fallback took 10.9 s at 8,000 edges).

### Added — `sd_grow_limit_set` / `sd_grow_limit_get`, bounded by default (`SD_GROW_LIMIT_DEFAULT` = 8 MiB)

⛔⛔ **THE TRADE THIS RELEASE MAKES, SAID OUT LOUD.** Lifting the cap turns "a path over 8,192 edges
renders wrong" into "a path over 8,192 edges costs whatever it costs" — and the scratch is
process-lifetime on an allocator with no `free()`, so ONE pathological path makes a process keep that
memory for life. MEASURED on this tree, rekha's hostile repro (4,096 maximally non-flat quads, one
`sd_canvas_fill_path` on 64x64 — the shape a font file can carry): **unbounded**, it grows to
1,048,576 entries and costs **150,602,312 B** of global heap, permanently. Under the default ceiling
the same fill keeps its first 262,144 edges, RECORDS the truncation, and costs **83,493,448 B** — of
which 50,135,040 B is the flatten mid-point waste rekha filed separately (below), not growth.
⇒ Over the ceiling sadish chooses **bounded memory and a flagged, degraded fill** over correct output
at any price; the flags are what keep the area engine off an open contour. `0` restores unbounded
growth; the setter returns the previous value so it can be scoped, like `sd_alloc_set`.
⚠ The ceiling bounds one REQUEST. Doubling abandons the old block, so a store that reaches it has also
paid for every smaller block below (8 MiB + 4 + 2 + … ≈ 16 MiB).

### Added — dashed strokes: `sd_canvas_stroke_path_dash` (new module `src/dash.cyr`)

SVG `stroke-dasharray` / `stroke-dashoffset` over the 0.6.0 styled stroker. `pattern` is a
caller-owned array of `n` 16.16 lengths (on, off, …), `offset` a 16.16 distance into it, any sign. An
ODD `n` is the array read twice; `pattern == 0`, `n <= 0` or an all-zero array is
`sd_canvas_stroke_path_ex` literally; a NEGATIVE entry or an overflowing period is
`SADISH_ERR_BOUNDS` with nothing drawn; the pattern restarts at every subpath.
⭐ **A dash is not a second stroker.** It is `_sd_sb_subpath` run on a PIECE of the flattened run, so
caps at both ends of every dash, joins inside one, the miter limit, the inside-a-curve join rule, the
clip, the `SD_AA_AREA` dispatch and one-call-one-fill all carry over. Both styled entry points now
share one verb walk.
⭐ **A closed subpath dashes around its closing point, and that meeting is a JOIN** — the first dash is
deferred and concatenated onto the last. MEASURED, a 24 px square at width 4, BUTT/MITER, [16,16] at
offset 8: the outer corner reads 255 through the join and 0 with the two dashes stroked apart.
⛔ **A dot is a zero-length ENTRY, not a dash the path ran out of.** Where a boundary falls exactly on
an open subpath's last vertex, the entry has positive length and no path left: nothing is drawn.
`"M 2,4 L 18,4"` under [4,12] is one 4 px dash and a 12 px gap — not a dash and a cap-sized blot at
x = 18. ⚠ Both of these were defects the review caught: the deferred head was stroked through a
recycled cut-point record (a line straight across the shape's interior), and the end-of-path dot
painted a full disc inside a declared gap.
⚠ Dash lengths follow the FLATTENED polyline — a chord-length under-estimate of arc length, so dashes
run long on a curve by about s/(3R): MEASURED at tol = 0.25 px, a radius-20 px circle is 0.63 % short.
Truncation does not creep (`sd_isqrt` floors one-sidedly): 900 raw units over 1,000 diagonal segments
= 0.0137 px, and a full pixel needs 65,536.
**Allocation:** 73,792 B on the first dashed stroke, then 0 B a call. A process that never dashes pays
nothing.

### Added — focal radial gradients and gradient transforms (`src/paint.cyr`)

`sd_gradient_radial_focal(cx, cy, r, fx, fy)` — SVG 1.1 `fx`/`fy` with focal radius 0. A focal point
on or outside the circle is moved inside to `r - ceil(r/2^10)` along its own direction. A focal point
at the centre is `sd_gradient_radial` byte for byte.
⭐ **`t` is EXACT** — `floor(t·2^16)` — for `t < 256`, which is every pixel inside the circle and,
under PAD, every pixel: the root needs ~113 bits, so it is estimated in i64 and the ~2.4 % of pixels
landing near a `2^-16` boundary are settled by the exact sign of the polynomial in 128 bits.
`sd_gradient_set_matrix(g, m)` — SVG `gradientTransform`; `m` maps gradient space to canvas space and
is copied. ⭐ A transformed LINEAR gradient is still a canvas-space linear gradient, resolved once per
blit in 128-bit arithmetic and painted by the same exact DDA, so it costs the same 14.0 ns/px.
⭐ Translations, quarter-turns, 2^k scales and mirrors are byte-identical to moving the geometry;
otherwise MEASURED under `rotate(30°)·scale(2)`, `floor(t·2^16)` is off by 1 at 7 linear / 20 radial /
19 focal pixels of 2,209, never more.
⛔ **Two matrices make the gradient render NOTHING**, both returning `SADISH_ERR_BOUNDS`: a singular
one (SVG's own rule) and one with any entry outside ±16,384.0 (sadish's precision bound, which refuses
perfectly invertible matrices such as a 16,384 px translation). Move the geometry, not the matrix —
and read the return code, because an ignored one leaves an invisible paint.
Also new: `sd_matrix_invert(m)` in `src/geom.cyr` (0 when singular), exact for identity, translations,
2^k scales and quarter-turns. ⚠ `SD_GRADIENT_CAP_OFFSET` is gone — the stop capacity is derived and
`+64` now holds an optional 72 B extension; the object stays 88 B, a plain gradient still costs
exactly 4,256 B, and **a blit still allocates 0 B**, first call included.

### Added — premultiplied writers, and gradient paint into them (new module `src/premul.cyr`)

`sd_clear_premul`, `sd_fill_rect_premul`, `sd_canvas_blit_premul_at` / `_premul`,
`sd_surface_rgba_at`, and `sd_canvas_blit_paint_premul_at` / `_premul`. These are the first sadish
writers whose output has REAL alpha: a transparent clear, a translucent rect, an anti-aliased edge over
a transparent surface, and a gradient panel with AA rounded corners over a transparent window — what a
`SETU_SURF_PREMULTIPLIED` surface needs so agnos composites it with `gpu_shader_op` #92 op 0x01.
⛔ **Alpha is an explicit `a`, and 0 means TRANSPARENT everywhere in the module** — the colour's alpha
byte is never read. #92 reads byte 3 literally, and misreading it is what produced the over-bright
ghosts this stack has paid for twice. Porting a `sd_canvas_blit_at` call means `a = 255`, not
`sd_alpha_of(c)`.
⭐ The arithmetic is agnos's own `cov_ref_px`, with `sd_premul`'s colour. Every output is valid
premultiplied (`c <= a`), swept over all 32,896 valid destination pairs and every (a, cov). Against an
f32 emulation of the kernel shaders: op 0x01 **0 mismatches over 8,421,376 inputs**; op 0x02 worst ±1.
⭐ **ONE EVALUATION, TWO COMPOSITORS**: the straight and premultiplied gradient blits are the same
function with a flag picking the store — same DDA, same roots, same spread, same ramp.
`sd_canvas_blit_paint_at`'s bytes are unchanged, and an OPAQUE gradient at full coverage paints
byte-identical pixels through both.
⚠ With a TRANSLUCENT stop the two roundings compound: MEASURED over ALL 4,261,478,400 lane inputs, the
signed difference premul − straight lies in **−2..+2**, narrowing to −1..+1 at full coverage — not to 0.
⚠ Stop alpha is interpolated STRAIGHT and premultiplied AFTER, per pixel (SVG's rule). A premultiplied
ramp would drag every transition toward the opaque stop: MEASURED, 63 levels at the midpoint.
⚠ A gradient still cannot express a fully transparent stop — the legacy rule spends alpha 0 on OPAQUE.
⚠ Repeated compositing of one translucent layer drifts: alpha stalls below 255 for a < 128, and 20
fills of grey 200 at a = 10 give 98 against an analytic 110.14. A single layer stays within 1.

### Fixed — a clip mask is PACKED `w*h`, on a canvas of any stride

Closes `docs/development/issues/2026-09-15-clip-masks-written-packed-but-read-by-canvas-stride.md`.
The mask was written `y*w + x` by all three producers and read `py*stride + px` by all three consumers
(`sd_fill_impl`, `_sd_area_fill`, `_sd_sb_flush`); they now read it packed. ⛔ Byte-identical — every
canvas `sd_canvas_new` makes has `stride == width`, so no caller can tell.
⚠ **The contract, now stated in the `SdCanvas` layout comment and at each reader:** the COVERAGE buffer
is addressed by the STRIDE and may one day be a window into a wider buffer; the CLIP MASK is a
sadish-owned `w*h` block and is addressed PACKED, always.
Also fixed next door: `sd_canvas_clear` zeroed one flat `w*h` run from the coverage pointer, which on a
wrapped canvas clears neither all of the window nor only the window; it now walks rows at the stride.
Byte-identical on every canvas `sd_canvas_new` makes.

### Filed by rekha — for 0.7.1, not fixed here

- `docs/development/issues/2026-09-15-flatten-keeps-subdividing-and-allocating-after-the-output-cap-is-full.md`
  — `sd_flatten_quad` / `_cubic` keep subdividing (and allocating mid-points) past the output cap.
  ⚠ **This release changes the shape of that cost, and not only for the better.** RE-MEASURED on this
  tree with the same repro: 0.6.0 emitted 8,192 points for 50,200,616 B; 0.7.0 emits all 1,048,577 for
  **83,623,976 B** in **3,133,451** allocations. Truncation is gone; the waste is not. rekha's own note
  anticipated it — "whatever replaces the cap should still bound total flatten work".
- `docs/development/issues/2026-09-15-path-construction-stores-through-a-refused-allocation.md`
  — `sd_path_new` / `sd_point_new` / `sd_path_flatten` store through unchecked `sd_alloc` results, so a
  hook that refuses faults inside sadish (SIGSEGV, rc 139).
- `docs/development/proposals/2026-09-15-path-capacity-for-known-size-paths.md` — `sd_path_new_cap`.

### Verified

All **25** `programs/*_test.cyr` pass: the 19 pre-0.7.0 suites (assertions unedited but for `area_test`
group N) plus `grow_edges_test` (216), `paint_focal_test` (144), `premul_test` (148),
`dash_test` (196), `clip_pitch_test` (106) and `paint_premul_test` (95). `fmt --check` clean, `lint` 0
warnings, `vet` clean, `distlib` in sync with 362 top-level names and none defined twice.
`sadish_version()` → **700**; `dist/sadish.cyr` 170,626 → 297,263 B; DCE smoke binary 16,328 → 16,728 B.
⭐ **Every item was built, adversarially reviewed, fixed and independently re-verified, and each proves
its tests by mutation.** The review found two real dash defects (above) that no suite had caught, and a
clip reviewer found six coverage loads no canvas could reach — correct code that could be pushed to the
wrong index with every suite still green; `clip_pitch_test` groups H and J close them with no source
change. Mutation totals this release: grow 102 (85 caught), paint2 139 (125), premul 72 (63), dash
(groups A–S, every reviewer defect pinned), clip (each reader reverted alone fails only the new suite),
paintpremul (per-pixel clipping pinned against an independent reference after a deleted `+ x0` left all
23 suites green).

## [0.6.0] - 2026-09-15 — the roadmap's three: styled strokes, gradient paint, exact coverage

The three items the README has listed as **next** since 0.4.0 — miter/bevel joins + butt/square
caps, radial + multi-stop gradients, full 2-axis signed-area coverage — plus the open 0.5.5 stride
issue. ⛔ **Everything a 0.5.5 caller already does is byte-identical**: the 14 pre-existing suites
pass with their assertions unedited, agnos's `tests/gpu/refagree.cyr` still prints **BYTE-IDENTICAL on
all 200 paths** against this `dist/`, and rekha (19 suites) and dhancha (18 suites) pass against it.

### Added — `sd_canvas_stroke_path_ex(cv, path, width, cap, join, miter_limit)` (new module `src/stroke.cyr`)

`SD_CAP_BUTT | SD_CAP_ROUND | SD_CAP_SQUARE` × `SD_JOIN_MITER | SD_JOIN_ROUND | SD_JOIN_BEVEL`, and
`miter_limit` as a 16.16 SVG `stroke-miterlimit` ratio (past it a join bevels; AT it, still a miter;
clamped to [1.0, `SD_MITER_LIMIT_MAX` = 1024.0]). An unknown cap or join returns `SADISH_ERR_BOUNDS`
and draws nothing. Open subpaths get caps, closed ones a join at every vertex and no caps; coincident
points are dropped; a lone point draws a disc (ROUND), a 2·hw square (SQUARE) or nothing (BUTT); a
180° reversal is a flat end; a drawing verb after CLOSE restarts at the moveto point, as the fill
reads it.

⛔ **ROUND/ROUND IS `sd_canvas_stroke_path`, UNCHANGED.** The 0.4.0 stroker moved from `raster.cyr`
to `stroke.cyr` verbatim (raster's diff there is a pure deletion) and `_ex(…, ROUND, ROUND, any)`
returns it — asserted against a verbatim 0.5.5 copy across 45 canvases. ⚠ So ROUND/ROUND alone keeps
its 0.4.0 after-CLOSE geometry, per-pass MAX seams, and 34,432 B of paths per closed rect on the seam.

⭐ **Styled pieces are ONE fill, not a MAX union of passes.** A per-piece MAX union is not a union:
two rects meeting at x = 4.5 leave the seam pixel at 127 (MEASURED); as one nonzero fill, 255. Rects,
join triangles, discs and square caps go into a process-lifetime edge batch, all wound the same way,
filled by a stroke-private walk with the default fill's sampling that merges overlapping spans and
bridges 1-LSB touches. Solid pixels reading 254 inside a stroke, fill_union vs the walk: width-8
cubic MITER 120 → 2, ROUND joins 282 → 2, a 10-vertex fractional polyline 36 → 0.
⚠ **Inside one curve's flattening the join style does not apply**: a flattened vertex mitres only
while its tip stays within 0.25 px of the disc, then rounds. At SVG's default limit 4 the cusp of
`M 10,30 C 50,5 10,5 50,30` drew a 6 px spike on a 4 px stroke (MEASURED); now it is round.
⚠ **The limit test is exact integer math** (directions rescaled to [2^13, 2^14), every intermediate
< 2^62); the suite asserts both sides of a ratio of exactly 1.25 and of √298/3.
⚠ **Batches hold 8,192 edges** (~800 miter segments). A longer subpath flushes mid-way and re-emits
the stretch around the seam, so seams ALONG a path stay within 1 level; ⚠ but where a subpath longer
than a batch CROSSES itself, pieces from different batches meet by MAX — MEASURED on 60 random
700–1,100-segment self-crossing walks: 22 split, worst 101 levels.
**Allocation:** 270,592 B once, from the GLOBAL `alloc`, at the first styled stroke (never the hook);
then **0 B** per straight-line styled stroke, hooked or not.

### Added — gradient paint: `SdGradient` + `sd_canvas_blit_paint_at` (new module `src/paint.cyr`)

```
var g = sd_gradient_linear(x0, y0, x1, y1);       # or sd_gradient_radial(cx, cy, r) — 16.16, canvas space
sd_gradient_add_stop(g, 0, sd_rgb(255, 0, 0));     # any order; equal offsets keep insertion order
sd_gradient_add_stop(g, SD_ONE, sd_rgba(0, 0, 255, 128));
sd_gradient_set_spread(g, SD_SPREAD_REFLECT);      # SD_SPREAD_PAD (default) | _REPEAT | _REFLECT
sd_canvas_blit_paint_at(cv, surface, g, dx, dy);   # clips every side, rows by sd_surface_stride
```

⚠ **Sampled at pixel CENTRES** with 16.16 geometry — the legacy `sd_canvas_blit_gradient` samples the
top-left corner with whole-pixel endpoints and is kept byte-identical; they are not interchangeable.
⭐ **t is exact per pixel, not accumulated**: linear t is stepped DDA-style with its remainder, radial
t takes its root to 16 fraction bits. Stops resolve into a 1025-entry ramp, and any ramp cell with a
stop strictly inside it is evaluated from the exact t — so a hard stop lands on the right pixel at any
axis length (a 0.3 stop on a 4,096 px axis flips at pixel 1229) and a 3.5 px band keeps its 4 pixels.
⚠ Stops use the legacy alpha rule (byte 0 = opaque) and interpolate straight alpha; source alpha =
coverage × paint alpha / 255, then `sd_canvas_blit_at`'s src-over. A one-stop opaque gradient paints
exactly `sd_canvas_blit_at`'s bytes. Following SVG: zero stops paint nothing; a zero-length axis or
r <= 0 paints the last stop.
⚠ **Precision bound**: gradient coordinates within ±16,383 px, verified exactly on 4,096 px rows at
the extremes. Outside it colours may be wrong; the ramp index is always clamped.
⭐ **A blit allocates nothing, first call included.** A gradient costs 4,256 B through `sd_alloc`
(object 88 + stops 64 + ramp 4,104); a stop-block doubling adds its new size.
MEASURED, 256×256, ns/px: linear 14.2, radial 44.3 (legacy 2-stop 13.7, solid blit 7.0).

### Added — `SD_AA_AREA`: exact 2-axis coverage (`sd_canvas_set_aa` / `sd_canvas_aa`, new module `src/coverage.cyr`)

The fill has been exact in x since 0.4.0 but SAMPLES y at 4 sub-scanlines: an edge at y = 4.3 reads
**63** (true value 76), at 4.7 **191** (178). `SD_AA_AREA` is signed-area cell accumulation (the
font-rs / stb_truetype v2 / FreeType "smooth" family) and reads 76 and 178.

⛔⛔ **OPT-IN, PER CANVAS; THE DEFAULT IS THE 0.4.0 LOOP, UNTOUCHED** — agnos's GPU rasteriser is gated
byte for byte on it. `sd_canvas_set_aa(cv, SD_AA_AREA)` applies to every fill into that canvas:
`fill_path`, `fill_union` (so the round stroker), the mask `clip_push_path` builds (it inherits the
parent's mode), and — through `_sd_sb_flush`'s dispatch — styled strokes.
⛔ **No streaks.** Every piece deposits exactly its height (`dy - a` into its cell, `a` into the next)
and heights come from exact endpoints, so right of a closed contour the running sum is exactly 0 —
asserted to the raw unit (8,004 edges in one pixel row). ⛔ An AREA fill whose edge list reaches
`SD_FLATTEN_CAP` may have been truncated (an open contour would streak), so it runs the default loop.
⚠ Both rules fold the INTEGRAL of the winding over a pixel: exact where a pixel's winding is {0, ±1},
the standard cell approximation elsewhere (a pixel-sized bow-tie reads 0).
⚠ **`SdCanvas` grew 40 → 48 B** (`SD_CANVAS_AA_OFFSET = 40`); `sd_canvas_new` stores the default
explicitly. Nothing in dhancha, rekha or agnos builds a canvas by hand. ⚠ **Every canvas costs 8 B
more on `sd_alloc`** — dhancha's `text_arena_test` passes unchanged but its printed figures move:
MEASURED against dhancha's current tree, a 12-label frame 415,280 → **415,368 B**, a 10-char label
38,016 → 38,024 B, the 64 KiB-arena spill 349,752 → 349,840 B. Its README/CHANGELOG figures want
re-measuring when it bumps its sadish pin.
MEASURED, 256×256 incl. flattening: curved blob **0.48 ms** per fill (SUBSCANLINE 2.6 ms); 400-edge
self-intersecting polygon 2.2 ms (35 ms); 8,000 edges 41 ms (10.9 s). **Memory:** one (w+1)-slot
accumulator from the global `alloc` on the first AREA fill (264 B at 32 px); **0 B** after.

### Fixed — every surface reader and writer addresses rows by `sd_surface_stride`

Closes `docs/development/issues/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md`.
`sd_plot`, `sd_line`, `sd_hline`, `sd_vline` (and so `sd_rect` / `sd_fill_rect` / `sd_clear`),
`sd_blend_hline` / `sd_fill_rect_blend`, `sd_surface_pixel_at`, `sd_canvas_blit_gradient`, the
presenter's row copy, and `sd_surface_write_ppm` (which the issue missed — it read `w*h` pixels as one
flat run) now load the stride once per call. Packed surfaces are byte-identical (a 66,558 B dump of
every primitive, a PPM and four presenter frames diffs empty against 0.5.5).
⚠ `sd_put(px, w, h, x, y, color)` keeps its signature and stays the PACKED form — a raw pointer
carries no stride; nothing in sadish or its consumers calls it any more. `sd_plot` / `sd_line` use a
private row-pointer store; `sd_line` steps the row pointer rather than re-multiplying (a per-pixel
`px + y*stride` MEASURED ~3.8 % slower; stepped, within noise of 0.5.5).

### Filed

`docs/development/issues/2026-09-15-clip-masks-written-packed-but-read-by-canvas-stride.md` — the
clip mask is written `y*w + x` and read `py*stride + px` by the fill, the area engine and the styled
flush. Unobservable today (every canvas has `stride == w`).

### Changed

- `sadish_version()` → **600**. `dist/sadish.cyr` 85,815 → 170,626 B; DCE smoke binary 16,000 → 16,328 B.
- Module order: `coverage.cyr` before `raster.cyr`; `stroke.cyr`, then `paint.cyr`, after it.

### Verified

All **19** `programs/*_test.cyr` pass — the 14 pre-existing suites with assertions unedited, plus
`stride_test` (103 checks; 58 fail against the 0.5.5 sources), `stroke_style_test` (193),
`paint_test` (248), `area_test` (163) and `integration_test` (33: the process's first-ever coverage
work is an AREA styled stroke under an arena hook — exactly 860,688 B on the global heap, **0** on
the arena, 0 / 0 warm; styled strokes on an AREA canvas read 51 / 25 where the walk reads 63 / 31;
clip + MAX through that dispatch; paint through AREA coverage onto a wrapped surface equals
`sd_canvas_blit_at`; the version probe). `fmt --check` clean, `lint` 0 warnings, `vet` clean,
`distlib` in sync. No new top-level name collides with anything in dhancha, rekha, crab or
agnos/tests/gpu.
⭐ **Every item was built, adversarially reviewed, fixed and independently re-verified, and each
proves its tests by mutation**: stride 22 mutations, all caught; stroke 103 (89 caught, 3 kill the
suite, 11 equivalent); paint 114 (110 caught, 4 equivalent); area 75 (68 caught, 7 equivalent or
speed-only); integration 3, all caught (no AREA dispatch: 8 checks; its guard inverted: 2; the
accumulator on the hook: 2).
⚠ **One pre-release rename**: the stroke item defined both `var _sd_sb_cap` (batch capacity) and
`fn _sd_sb_cap` (draw a cap) — the fn is now `_sd_sb_endcap`. Duplicate names shadow silently here.

## [0.5.5] - 2026-09-14 — a rasterizer that can be told where its memory comes from

### Added — `sd_alloc` / `sd_alloc_set` / `sd_alloc_get` (new first module `src/alloc.cyr`)

⭐⭐ **The allocation seam.** 23 of the 32 `alloc(` sites sadish had (error 1, geom 2, path 8,
present 6, surface 2, raster 4) now go through `sd_alloc(n)`; the other 9 — `sd_fill_impl`'s
ectx/edges/fbuf/fctx/cross/accrow and the stroker's run/fbuf/fctx — became the process-lifetime
scratch in `_sd_scratch_init` / `_sd_fill_accrow_for`, deliberately on the global `alloc` (⛔
below). A consumer may install a hook:

```
var prev = sd_alloc_set(&my_arena_hook);   # fn(n): ptr, 0 on failure; returns the PREVIOUS hook
... draw ...
sd_alloc_set(prev);                        # 0 restores the global allocator; sd_alloc_get() reads it
```

`lib/alloc.cyr` is a bump allocator with **no `free()`**, so until now every canvas, path, point
and clip mask sadish made was permanent. dhancha's frame arena (`dh_falloc`) is the only way a
rendered frame can cost the heap nothing, and dhancha could not route sadish onto it — the defect
it filed as `2026-09-13-scalable-text-allocates-per-call-outside-the-frame-arena.md`, and one of
the two blockers on **crab** adopting a proportional face (crab's headline gate asserts a rendered
frame costs the global heap **exactly 0 bytes**). This is the sadish half of closing it.

⚠ **The hook applies to EVERYTHING sadish allocates while it is set** — surfaces, canvases,
coverage buffers, clip masks, paths, verb/point arrays, points, matrices, flatten mid-points, the
PPM/presenter buffers. sadish does not distinguish the result from the working set; the consumer
scopes the hook around the work whose lifetime it is choosing. ⇒ Nothing returned under an arena
hook may be kept across that arena's reset.
⚠ A hook returning 0 is handled exactly as `alloc()` returning 0 already was. No new failure path.

### Added — `sd_canvas_blit_at(cv, surface, color, dx, dy)`

Composite the canvas with its (0,0) at surface pixel `(dx, dy)`; either may be negative or beyond
the surface, and the blit clips to the surface on every side. Same src-over math as
`sd_canvas_blit`, verbatim — which is now `sd_canvas_blit_at(cv, s, c, 0, 0)`.

⚠ **Rows are addressed by `sd_surface_stride`, not `width*4`.** The pre-0.5.5 `sd_canvas_blit`
used the surface WIDTH as its row pitch — right for every packed surface sadish makes itself, and
one row of shear per row on a WRAPPED surface (dhancha's `dh_surface_wrap` over a framebuffer or a
pane sub-rect, where `stride != width*4`). The suite builds such a header by hand and asserts row
1 lands at the pitch (128 B), not at `width*4` (32 B).
⚠ **This is now the ONE writer that honours the stride.** `sd_put`, `sd_surface_pixel_at`,
`sd_hline`, `sd_vline`, `sd_blend_hline`, `sd_canvas_blit_gradient` and the presenter's row copy still
address by `width * 4` — right for every packed surface sadish makes, one row of shear per row on a
wrapped one, MEASURED as 4,000 padding pixels overwritten by a `WINDOW` background under text that
landed straight. Filed as
`docs/development/issues/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md`.

### Changed — the fill/stroke scratch is process-lifetime, allocated once

⭐ `sd_fill_impl` allocated `ectx + edges + fbuf + fctx + accrow + cross` FRESH on every call, and
`sd_canvas_stroke_path` its `run + fbuf + fctx` — all sized to the fixed `SD_FLATTEN_CAP`, none of
it dependent on the path. **MEASURED on 0.5.4** (8x8 canvas, one call, `alloc_used()` delta):

| call | 0.5.4, every call | 0.5.5, first call | 0.5.5, every call after |
|---|---|---|---|
| `sd_canvas_fill_path`, 4-line rect | 327,824 B | 589,960 B | **0 B** |
| `sd_canvas_fill_path`, 1 quad + lines | 328,016 B | — | 144 B (9 flatten mid-points) |
| `sd_canvas_fill_path`, 760x300 triangle | 333,856 B | 6,080 B (accrow) | **0 B** |
| `sd_canvas_stroke_path`, closed 4-vertex rect | 2,789,016 B | — | 34,432 B (8 paths + points) |
| one frame of fill+union+stroke+clip+blit (16x16) | 3,456,048 B | — | **0 B** under an arena hook (45,000 B on the arena) |

The scratch is now one process-wide set — `_sd_fill_*` and `_sd_stroke_*` module globals in
`raster.cyr`, allocated lazily ONCE by `_sd_scratch_init()` (MEASURED 589,896 B: fill 458,800 +
stroke 131,096) plus a per-row accumulator that re-allocates only when a WIDER canvas than ever
seen arrives (w*8: 64 B at 8 px, 6,080 B at 760). Every per-call reset (edge count, flatten ctx)
is exactly as before, and the output is **byte-identical**: all 13 pre-existing suites pass
unchanged, and a probe dumping coverage for a rect, a quad, a cubic and a stroke diffs empty
against 0.5.4.

⛔ **From the GLOBAL `alloc`, never the hook.** A consumer's hook is typically an arena that gets
reset every frame; a scratch drawn from it would be dangling on the second frame. So the scratch
is the one thing sadish allocates that the seam does not see — its one-time cost lands on the
global heap at the first fill, and never again.
⛔ **Two sets, not one.** `sd_canvas_stroke_path` holds `run` live across the `sd_canvas_fill_union`
calls `sd_stroke_run` makes; the stroker's `run/fbuf/fctx` are its own globals so non-aliasing is
visible in the names rather than reasoned about per call.
⚠ **Single-threaded entry points, said out loud.** The rasterizer never had a lock anywhere and was
never thread-safe; a shared scratch makes `sd_fill_impl` / `sd_canvas_stroke_path` explicitly so.
Checked: every consumer (dhancha, rekha, agnos's refagree test) rasterizes from one thread.
The flatten MID-POINTS stay on the seam: proportional to the curve work, not fixed-capacity, so the
hook decides where they live.

### Changed — toolchain `6.6.2` → `6.6.4`

Bumped before any source change; all 13 suites passed on the new pin first. `sadish_version()`
now answers **505** (it had read 401 since 0.4.1). `dist/sadish.cyr` 75,450 → 85,815 B; the DCE
smoke binary 15,912 → 16,000 B.

### Verified

All **14** `programs/*_test.cyr` pass (13 existing + the new `alloc_test`, 96 numbered checks).
`fmt --check` clean, `lint` 0 warnings, `vet` clean, `distlib` in sync.
⛔ **The process's FIRST fill runs under an arena hook** (`alloc_test` group A). With no hook
installed `sd_alloc` IS `alloc`, so a warm-up fill outside a hook lands the scratch on the global
heap under either spelling and cannot see the ⛔ above: an earlier draft of the suite warmed up
unhooked, and all nine scratch sites flipped `alloc` → `sd_alloc` passed it 90/90. The gate is now
the pair of exact figures — 590,152 B (589,896 scratch + 32*8 accrow) on the global heap and
EXACTLY 0 on the arena — followed by a wider (64 px) canvas under the same hook whose accrow
re-grow is exactly 512 B on the heap and 0 on the arena.
⭐ **Fifteen mutations, each of which fails the suite** (failed checks in brackets): a raster
`sd_alloc(` back to `alloc(` for the coverage buffer [2] and for the clip node [2]; `sd_point_new`
off the seam [3]; fill edges + cross back to per-call `alloc` [8]; the stroke run per-call [3];
the accrow capacity never remembered [3]; `blit_at` ignoring `dy` [8]; `blit_at` using `width*4`
as the pitch [4]; `blit_at` not clipping on the left [2, then the process faults — a mis-clipped
column writes into the surface header's stride field]; not clipping on the right [3];
`sd_alloc_set` returning the new hook instead of the previous [8]; `sd_alloc` ignoring the hook
[4]; all nine scratch sites `alloc` → `sd_alloc` [4, then the process faults in group B — the
scratch dangles after the first `arena_reset`, which is the corruption the ⛔ exists to prevent];
the accrow site alone [4]; the eight fixed-scratch sites alone [2, then faults].

## [0.5.4] - 2026-09-11

### Changed

- **Toolchain `6.5.36` → `6.6.2`.** No source change. Zero compiler rejections,
  zero fail-open sites. All `programs/*_test.cyr` RUN suites pass.

## [0.5.3] - 2026-08-31 — a fill that reads its destination

### Added — `sd_fill_rect_blend` / `sd_blend_hline`

⭐⭐ **The first fill in sadish that READS its destination.** Every other span writer stores four
bytes per pixel and never loads — correct and fast for opaque paint, and exactly why a translucent
veil was impossible. dhancha wanted a modal backdrop for its 0.9.23 sheet and had to ship a
**scanline dither** instead; this is what that was waiting on.

```
sd_fill_rect_blend(s, x, y, w, h, color, a)     # a = 0..255 coverage
sd_blend_hline(s, x0, x1, y, color, a)          # the span workhorse
```

⛔⛔ **`a` IS AN EXPLICIT ARGUMENT AND 0 MEANS FULLY TRANSPARENT — THE OPPOSITE OF `sd_alpha_of`.**
The legacy packed-alpha rule maps a 0 byte to **opaque**, which exists so a bare `sd_rgb` value is
not invisible. Carrying that convention into a blend would make `sd_fill_rect_blend(..., 0)` paint
**solid** — the single most surprising outcome available, and the exact trap that made
`sd_rgba(0, 0, 0, 128)` paint black rather than a half veil. ⇒ Alpha is passed separately, it is
never read out of `color`, and 0 is a no-op. The test asserts that in both directions.

⚠ **Source-over, integer, per channel**: `out = (src*a + dst*(255-a)) / 255`. Rounding truncates, so
two 50 % veils are not identical to one at 75 % — true of any integer compositor and not worth a
fractional pixel format to fix.
⚠ **The destination alpha byte is left at 255.** sadish surfaces are opaque render targets; a blend
that also composited alpha would make the result translucent to whatever draws it next, and every
consumer here presents to a screen.

### Changed — toolchain pin 6.5.27 → 6.5.36

⚠ **Nine releases stale.** Per the standing rule a repaired repo does not stay on an old pin; `lib/`
re-vendored with `cyrius lib sync`.

### Verified

All **14** `programs/*_test.cyr` pass (13 existing + the new `blend_test`). `fmt --check` clean,
`lint` 0 warnings.
⭐ **Three mutations, each of which fails the suite**: `a == 0` treated as opaque (the legacy trap);
the blend ignoring its destination and storing the source directly; and the red channel reading the
blue source, which only shows on a non-grey blend.

## [0.5.2] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.5"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **THE ARTIFACT CHANGED, so this is not a cosmetic edit.** The build went `118032 -> 126448` bytes and the binary differs. The pin is not a comment: it selects the stdlib snapshot under `~/.cyrius/versions/<pin>/lib`, so moving it swaps the library code this repo compiles against.

⚠ The vendored `lib/` was re-synced to the 6.5.27 bundled set, which clears the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [0.5.1] - 2026-08-02

### Changed — cyrius pin 6.4.71 -> 6.5.5

Toolchain catch-up across the whole desktop stack, cut together so the next burn runs binaries built
by ONE compiler rather than 6 different ones.

⚠ **The pin was documentation, not enforcement.** `cyrius build` compiles with the INSTALLED `cycc`,
prints a `toolchain drift` warning, and carries on — so this project was already being built by 6.5.5
before this bump. Verify provenance with `~/.cyrius/versions/<pin>/bin/cyrius` when it matters.

⭐ What the gap actually contained, for a reader deciding whether to care:
- **6.5.1** made overload-suffix arity a hard **error** where it used to warn. Latent arity
  mismatches are now build failures instead of silently-wrong code — good, and the reason this
  sweep surfaced real defects elsewhere in the stack.
- **6.4.75** fixed `fn_table` growth past 8192 silently corrupting six fn-indexed side tables.
- **6.5.0** added file-scoped `private` / per-item `public` — the first real answer to this
  ecosystem's duplicate-`fn`-silently-shadows hazard.
- **6.4.82** completed the agnos GPU syscall wrapper band to `#82`-`#95`, so `sys_gpu_shader_op`
  (#92) and `sys_gpu_modeset_op` (#93) no longer need a raw `syscall()` behind an `#ifdef`.

### Verification

Host + `--agnos` builds green; **12 RUN tests** pass; `distlib` regenerated.

## [0.5.0] - 2026-07-23

### Added — a real ALPHA channel: `sd_rgba` · `sd_alpha_of` · `sd_color_a` · `sd_premul`

Until now every primitive hardcoded the alpha byte to **255 in four separate places** and the colour type
had no alpha at all. Invisible, because nothing downstream read byte 3 — but agnos's `gpu_shader_op` **#92**
op 0x01 does: it performs **premultiplied src-over on the GPU shader cores**, the one thing a CPU blit
cannot do cheaply. Without a real alpha channel that whole hardware-blend path had **no producer**.

`sd_put`, `sd_hline` and `sd_vline` now write the colour's alpha instead of a hardcoded 255. (`sd_fill_rect`
and `sd_clear` route through `sd_hline`, so fixing `sd_put` alone would not have covered fills — the three
sites were independent.)

⚠ **An alpha byte of 0 means OPAQUE, not invisible, and that is what makes this additive.** `sd_rgb(r,g,b)`
returns `0x00RRGGBB` with byte 3 = 0, and every existing caller and test depends on that value; treating 0
as transparent would make every pre-existing drawing vanish. `sd_alpha_of()` maps 0 → 255. The cost is that
fully-transparent is not expressible as a colour — the right trade, since it is the one alpha value with no
visible effect.

⚠ `sd_premul()` is the **sanctioned producer** for surfaces flagged `SETU_SURF_PREMULTIPLIED`. Passing
straight alpha to #92 does not error — it renders washed out, silently, with no diagnostic anywhere.

**Not changed:** `sd_canvas_blit`'s coverage path still flattens anti-aliased coverage into opaque RGB. That
is correct for an opaque destination and is not what blocked #92; a premultiplied coverage path would be a
new function, not a change to this one.

## [0.4.2] - 2026-07-23

### Changed — cyrius pin 6.4.25 → 6.4.71

Toolchain refresh across the draw stack. Materialised `lib/` re-synced (`cyrius lib sync --full`).
No source change; build + tests green at the new pin.

## [0.4.1] - 2026-07-08 — toolchain alignment

Pin/hygiene release — no code change; the 2D vector core is byte-identical to
0.4.0 (all RUN tests green).

### Changed

- **Cyrius pin `6.4.7` → `6.4.25`** — aligns sadish with the desktop stack
  (setu + dhancha, its consumer, both pin `6.4.25`) instead of drifting behind.
  Builds + all 13 RUN tests (aa / blit / clip / draw / fill / flatten / geom /
  gradient / grow / present / rotate / stroke / smoke) pass.
- `sadish_version()` → **401**.

## [0.4.0] - 2026-07-05

The full 2D vector core: skew (affine set complete), gradient paint, unbounded
paths, stroking, a clip stack, and an analytic coverage engine replacing the
supersampler. 12 RUN tests.

### Added
- **Matrix skew** (`sd_matrix_skew`, `sd_fixed_div`) — completes the affine
  transform set (identity / translate / scale / rotate / skew); shear angles
  via CORDIC `tan = sin/cos`. Verified in `programs/rotate_test.cyr`.
- **Linear gradient paint** (`sd_canvas_blit_gradient`) — composites coverage
  with the source color lerped along a linear axis from `c0` to `c1`
  (projection clamped to the endpoints), src-over onto an `SdSurface`.
  Verified in `programs/gradient_test.cyr`.
- **Growable path backing** (`sd_path_grow`) — `SdPath` is now unbounded: the
  verb + point arrays double on overflow instead of capping at
  `SD_PATH_CAP` (256). Verified in `programs/grow_test.cyr` (1000 verbs across
  several grows, data preserved).
- **Stroking** (`sd_canvas_stroke_path`, `sd_isqrt`, `sd_canvas_fill_union`) —
  strokes a path with round caps + round joins: a rectangle per flattened
  segment (perpendicular offset via integer sqrt) + a disc per vertex,
  MAX-unioned into the coverage so overlaps combine regardless of winding.
  Curves flatten first; subpaths + close are honored. Verified in
  `programs/stroke_test.cyr`. (Miter/bevel joins + butt/square caps + piece
  batching are TODO(v0.5).)
- **Clip paths / clip stack** (`sd_canvas_clip_push_rect` /
  `sd_canvas_clip_push_path` / `sd_canvas_clip_pop`) — a coverage-mask clip
  region held as a stack on the `SdCanvas`; push intersects a rect or a filled
  path into the active clip, fills + strokes are multiplied by the mask, pop
  restores. Verified in `programs/clip_test.cyr` (nested intersect + pop).

### Changed
- **Analytic coverage rasterizer** — the fill core (`sd_fill_impl`) now
  computes coverage ANALYTICALLY in x (exact horizontal span overlap via
  `sd_span_add`) across `SD_FILL_SS` vertical sub-scanlines, replacing the 4×4
  point supersampling. Smoother sub-pixel AA (no longer quantized to 17 levels)
  and much cheaper — edge crossings are computed per row, not per pixel. Every
  fill / stroke / clip inherits it; the whole suite stays green. Verified in
  `programs/aa_test.cyr` (0.3 and 0.7 edge coverage → 76 / 178, which the
  supersampler could not produce). Full 2-axis signed-area accumulation is
  TODO(v0.5).

## [0.3.0] - 2026-07-05

The rasterizer — the vector core comes alive. Paths now flatten, fill to
anti-aliased coverage, and composite onto a surface; plus affine rotation.
All five pieces are RUN-tested.

### Added
- **Signed fixed-point** (`sd_asr` + `sd_fixed_mul` routed through it, plus
  `sd_abs`) — Cyrius `>>` is logical (zero-fill); the signed arithmetic-right
  shift keeps negative products (mirrors, rotations, below-origin coords)
  correct. Guarded by `programs/geom_test.cyr`.
- **Adaptive Bézier flattening** (`sd_path_flatten`) — real de Casteljau
  subdivision for quad + cubic verbs, second-difference flatness vs `tol`,
  midpoints via `sd_asr`. Replaces the anchor-only stub.
  (`programs/flatten_test.cyr`.)
- **Scanline coverage fill** (`sd_canvas_fill_path`) — the headline: builds a
  closed-edge list from the flattened path (each subpath closed back to its
  moveto), then supersampled (4×4) anti-aliased coverage via a +x winding ray
  per sub-sample, honoring both nonzero and even-odd rules.
  (`programs/fill_test.cyr`.)
- **Coverage composite** (`sd_canvas_blit`) — retyped from the raw-`fb` stub to
  composite coverage × a solid color, src-over, onto an `SdSurface`.
  (`programs/blit_test.cyr`.)
- **Affine rotate** (`sd_matrix_rotate`, `sd_cos`/`sd_sin`) — sovereign
  fixed-point trig via CORDIC (16 iterations, shifts + adds only, no float / no
  libm), range-reduced to [-π/2, π/2]. (`programs/rotate_test.cyr`.)

### Notes
- Coverage is supersampled (4×4 → 17 levels); an analytic signed-area cell
  accumulator (smoother AA, lower cost) is TODO(v0.4), along with gradient
  paints, clip paths, stroking, matrix skew, and a growable path/flatten
  backing.

## [0.2.0] - 2026-07-05

The "easy 80%" lift — sadish stops being a scaffold. A real pixel surface,
the direct 2D primitives, and a present backend, all **adapted from the
ecosystem's framebuffer games rather than reinvented** (see
`docs/development/prior-art.md`), and all CI-gated. The anti-aliased coverage
rasterizer + adaptive Bézier flattening land in v0.3.0.

### Added
- **Pixel surface + direct primitives** (`src/surface.cyr`, `src/draw.cyr`) —
  `SdSurface`, a 32bpp BGRA pixel buffer (packed `0x00RRGGBB` colors, top-left
  y-down, `off=(y*w+x)*4`), plus the opaque integer-pixel primitives
  `sd_plot` / `sd_line` (integer Bresenham) / `sd_hline` / `sd_vline` /
  `sd_rect` / `sd_fill_rect` / `sd_clear`, and `sd_rgb` + channel extractors.
  Lifted from the ecosystem's framebuffer games (encom-hits `draw.cyr`
  Bresenham; the BGRA/present convention shared with cyrius-bb/-polyomino/
  -doom). Verified by `programs/draw_test.cyr`, a headless pixel-readback RUN
  test (exact-diagonal Bresenham, span clamping, filled vs hollow rect, silent
  out-of-bounds clip).
- **Present backend** (`src/present.cyr`) — `sd_surface_write_ppm` (binary P6
  PPM dump, platform-neutral file syscalls) + the Linux `/dev/fb0` `SdPresenter`
  (`sd_present_open`/`_blit`/`_close`: geometry probe, largest-integer-scale
  center-letterbox blit honoring the physical pitch, 32bpp + RGB565 paths),
  adapted from encom-hits' `engine.cyr`. Struct-based/reentrant (the encom
  original was module-global + fixed 320×240). Verified by
  `programs/present_test.cyr` (PPM write + readback); the fb0 path is never
  exercised in tests — it writes the live display. The AGNOS `blit`#39 sink is
  a future backend beside fb0.
- **CI + release** (`.github/workflows/{ci,release}.yml`) — build / lint / fmt /
  vet / dist-sync + RUN-test suites, a security scan (raw execve/fork/sys_system),
  and docs + version-consistency gates; semver-tag release that archives the src
  tarball + `dist/sadish.cyr` + `SHA256SUMS`. Toolchain pinned via
  `cyrius.cyml` (no version hardcoded in YAML).

## [0.1.0] - 2026-07-05

### Added
- Repo scaffolded: a buildable, link-clean pure-Cyrius skeleton for the
  low-level 2D vector graphics core — `SdPoint`/`SdMatrix` (16.16
  fixed-point geometry, live), `SdPath` (verb + point construction, live)
  with a `sd_path_flatten` stub, `SdCanvas` (coverage buffer) with
  `sd_canvas_fill_path`/`_blit` stubs, the `SadishErr` model, and a
  `programs/smoke.cyr` link-check. The scanline coverage rasterizer +
  adaptive Bézier flattening land in v0.3.0.
