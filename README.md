# sadish

Version: 0.9.0

**sadish** (सदिश — *sa* "with" + *diś* "direction" = "having direction":
the modern Sanskrit/Hindi word for **vector**; antonym अदिश *adish* =
scalar) is a pure-Cyrius, low-level **2D vector graphics core** for
AGNOS — the sovereign foundation everything vector builds up from.

It turns resolution-independent geometry into pixels: path construction
(move / line / quadratic + cubic Bézier), curve flattening, analytic
anti-aliased fill (even-odd + nonzero winding), stroking, affine transforms,
clipping, and solid + gradient paints → a pixel/coverage buffer that blits via
the kernel `blit#39` (or a host framebuffer under test). No GPU, no C shim, no
external binaries — everything-is-i64 integer arithmetic (16.16 fixed-point)
all the way down.

sadish is the **low-level lib you build up from**: `rekha` (font
outlines), a canvas/drawing API, compositing, SVG, and the `dhancha` UI
toolkit are all **consumers**, not re-implementations.

## Scope

- **v0.1.0 — scaffold.** Buildable, link-clean skeleton: real type shapes +
  public fn signatures, algorithm bodies stubbed. Superseded below.
- **v0.2.0 — surface + direct primitives + present (shipped).** The "easy
  80%" lifted from the ecosystem's framebuffer games (see
  `docs/development/prior-art.md`): `SdSurface` (32bpp BGRA), the opaque
  integer-pixel primitives (`sd_plot`/`sd_line`/`sd_hline`/`sd_vline`/`sd_rect`
  /`sd_fill_rect`/`sd_clear`), and the present backend (`sd_surface_write_ppm`
  + the Linux `/dev/fb0` `SdPresenter`).
- **v0.3.0 — the rasterizer (shipped).** The vector core comes alive — signed
  fixed-point (`sd_asr`/`sd_fixed_mul`), adaptive de Casteljau flattening
  (`sd_path_flatten`), scanline AA coverage fill (`sd_canvas_fill_path`,
  nonzero + even-odd), src-over composite (`sd_canvas_blit`), and CORDIC affine
  rotation (`sd_matrix_rotate`).
- **v0.4.0 — the full 2D core (shipped).** RUN-tested end to end:
  - **Affine set complete** — `sd_matrix_skew` (+ `sd_fixed_div`) joins
    translate/scale/rotate.
  - **Linear gradient paint** — `sd_canvas_blit_gradient`.
  - **Unbounded paths** — `sd_path_grow` doubles the backing on overflow.
  - **Stroking** — `sd_canvas_stroke_path`, round caps + round joins
    (`sd_isqrt` offsets + `sd_canvas_fill_union`).
  - **Clip stack** — `sd_canvas_clip_push_rect` / `_push_path` / `_pop`.
  - **Analytic coverage** — the fill core is now exact-in-x across vertical
    sub-scanlines (smoother + cheaper than the 4×4 supersampler).
- **v0.5.5 — the allocation seam (shipped).** `sd_alloc` / `sd_alloc_set` /
  `sd_alloc_get` route EVERYTHING sadish allocates through a consumer hook
  (dhancha's per-frame arena), the fill/stroke fixed-capacity scratch becomes
  process-lifetime (a straight-line fill costs the heap 0 B after the first
  call; was 327,824 B per call), and `sd_canvas_blit_at(cv, s, color, dx, dy)`
  places + clips a canvas on a surface, honouring `sd_surface_stride`.
- **v0.6.0 — the roadmap's three (shipped).** Everything a 0.5.5 caller does is
  byte-identical (agnos's refagree oracle: 200/200):
  - **Styled strokes** — `sd_canvas_stroke_path_ex(cv, path, width, cap, join,
    miter_limit)`: butt/round/square caps, miter/round/bevel joins, SVG
    `stroke-miterlimit` (`src/stroke.cyr`). Round/round is the 0.4.0 stroker.
  - **Gradient paint** — `sd_gradient_linear` / `sd_gradient_radial`, multi-stop,
    pad/repeat/reflect, `sd_canvas_blit_paint_at` (`src/paint.cyr`).
  - **Exact 2-axis coverage** — opt-in per canvas with
    `sd_canvas_set_aa(cv, SD_AA_AREA)` (`src/coverage.cyr`); the default
    4-sub-scanline engine is unchanged.
  - **Stride-correct surfaces** — every primitive, blit, the PPM writer and the
    presenter address rows by `sd_surface_stride`.
- **v0.7.0 — dashes, focal gradients, real alpha (shipped).** All five 0.6.0
  candidates, byte-identical for everything a 0.6.0 caller does (refagree 200/200):
  - **Dashed strokes** — `sd_canvas_stroke_path_dash` (SVG `stroke-dasharray` /
    `stroke-dashoffset`) over the styled stroker (`src/dash.cyr`).
  - **Focal radial gradients + gradient transforms** — `sd_gradient_radial_focal`,
    `sd_gradient_set_matrix`, plus `sd_matrix_invert` (`src/paint.cyr`, `src/geom.cyr`).
  - **Premultiplied writers** — transparent/translucent clears, rects, coverage
    blits and gradient paint with REAL alpha, for `SETU_SURF_PREMULTIPLIED`
    surfaces agnos composites with `gpu_shader_op` #92 (`src/premul.cyr`).
  - **`SD_FLATTEN_CAP` stops being a cliff** — the edge list, crossings, stroke
    run, curve flags, styled batch and `sd_path_flatten` all GROW instead of
    silently truncating; a 20,000-gon disc filled 24.5 % of its ink in 0.6.0 and
    99.997 % now. ⚠ The trade: growth is process-lifetime on an allocator with no
    `free()`, so it is BOUNDED by default — `sd_grow_limit_set` /
    `sd_grow_limit_get`, `SD_GROW_LIMIT_DEFAULT` = 8 MiB a request; over it a fill
    degrades and RECORDS the truncation rather than growing without limit.
  - **Clip-mask pitch** — a mask is a packed `w*h` block on a canvas of any stride,
    stated in the layout comment and gated by `programs/clip_pitch_test.cyr`.
- **v0.7.1 — rekha's repairs (shipped).** All three filings closed, rendering
  byte-identical (refagree 200/200; rekha 23 suites and dhancha 18 green):
  - **Flattening allocates only what it emits** — the de Casteljau mids are plain
    locals now, so a hostile 4,096-quad flatten falls from 83,623,976 B / 3.13 M
    allocations to **3,076,136 B / 65,287**, and the recursion stops once the
    output is full.
  - **A per-operation flatten budget** — `SD_FLATTEN_BUDGET_DEFAULT` = 65,536
    points, `sd_flatten_budget_set` / `_get`, `sd_flatten_degraded`. ⚠ Through
    0.7.1 a fill opened one operation PER CURVE VERB, so the budget did not bound
    a whole fill; **0.7.2 scopes the fill** (see below).
  - **`sd_path_new_cap(n_verbs, n_points)`** — a path at a caller-known capacity,
    for consumers that know the size before the first moveto.
  - **A refused allocation is a return code, not a fault** — every `sd_alloc`
    result in path/geom/raster/present/error is checked and propagated, a refusal
    costs the caller nothing it had, and a starved fill returns `SADISH_ERR_OOM`
    instead of painting a wrong picture under `SADISH_OK`.
  - **A drawing verb before any moveto** no longer dereferences a null point
    (SIGSEGV on 0.7.0 and 0.6.0 alike); every walk skips such verbs, as SVG does.
- **v0.7.2 — the residue a 0.7.1 audit found (shipped).** Four filings marked
  closed turned out to have unmet asks of their own; this is them:
  - **Strokes survive a refusing hook** — `sd_canvas_stroke_path` / `_ex` /
    `_dash` return `SADISH_ERR_OOM` instead of faulting (SIGSEGV on 0.7.1) or
    under-drawing in silence; `src/alloc.cyr`'s seam contract now says what is
    true, including what still faults.
  - **A fill is ONE flatten operation** — the budget bounds a whole untrusted
    outline, not one curve of it: the hostile repro falls 16,711,680 B →
    **1,044,480 B** unwrapped. ⚠ Past the 65,536-point budget a fill now degrades
    its remaining curves to chords and says so; the largest legitimate fill
    measured anywhere here spends 40 % of it.
  - **`sd_path_new_cap` sizes verbs and points separately** — the proposal's
    78,656 B target for the ASCII glyph set, to the byte (5.51x less than
    `sd_path_new`). ⛔ `SD_PATH_RESERVED_OFFSET` is removed and `sd_path_grow`
    gained an argument.
  - **Three coverage loads and three shipped headers** that no suite could tell
    apart, or that stated the opposite of the code, are pinned and corrected.
- **v0.8.0 — the repair backlog closes (shipped).** ⛔ All four issue filings are
  archived; `docs/development/issues/` holds only its README:
  - **The stroker's piece paths open at their exact size** — `sd_path_new_cap`
    finally has an in-tree caller. A closed 8x8 rect stroke costs the seam
    34,432 B → **3,232 B**, a 54-glyph round-stroked label 16,357,904 B →
    **1,557,176 B**, both in an unchanged number of allocations (which is what
    proves no array doubled).
  - **A flattened polyline says whether it is whole** — `SdPolyline` is 24 B with
    a verdict word; `sd_polyline_truncated` / `_degraded` / `_verdict`. The prefix
    is flagged, not refused; "refuse" belongs at the draw, where it has been since
    0.7.1.
  - **The presenter's `open()` is reachable without a display** —
    `sd_present_open_fd` / `_path`, gated over a regular file with descriptors
    counted; it also gates three `memset`s that only a reused arena can see.
- **v0.9.0 — `SdPath` stores its points inline (shipped).** x at +0, y at +8, 16 B
  a slot, instead of 8 B pointers to 16 B `SdPoint`s. The ASCII glyph set goes
  78,656 → **58,672 B** and 2,783 → **285** allocations; a path is three
  allocations whatever it holds. New accessors `sd_path_point_x` / `_y` /
  `sd_path_verb_at` end the coupling that made this a breaking change at all.
  ⚠ `SD_GROW_LIMIT_DEFAULT` doubled to 16 MiB so the strokable path length is
  unchanged (a run point went 8 B → 16 B); the fill's edge bound doubles with it.
  ⚠ **rekha must port** — 4 of its 23 suites read the old layout and SIGSEGV;
  dhancha is unaffected. Filed with the port in
  `rekha/docs/development/issues/2026-09-16-sadish-0.9.0-inlines-path-points-…`.
- **later (candidates):** a premultiplied AGNOS `blit#39` fast path; rekha adopting
  `sd_path_new_cap` (`proposals/2026-09-15-path-capacity-…` item 3); and
  `sd_present_open`'s single remaining ungated line, which only a display can gate.

## Place in the stack

sadish is a **leaf** — no upstream sadish dependencies (only the Cyrius
stdlib). It is the bottom of the vector-graphics tower:

```
  dhancha (UI toolkit) ─┐
  SVG / canvas / compositing ─┤
  rekha (font outlines) ──────┴─▶ sadish  (this repo: surface + primitives + paths + rasterizer)
                                     │
                                     ▼
                          coverage buffer ─▶ blit#39 (AGNOS) / host framebuffer
```

Everything above consumes sadish's surface + path API and coverage buffer;
none of them re-implement rasterization.

## Consumers

- **rekha** — font outline rendering (glyph contours → sadish paths →
  coverage).
- **dhancha** — the AGNOS UI toolkit (widgets draw through sadish).
- A canvas / drawing API, compositing, and SVG layers (planned) sit on the
  same core.

Live consumers — rekha (≥ 0.3.10 routes its outline scratch and font
records through `sd_alloc`), dhancha (≥ 0.10.0 installs its per-frame arena
as the hook around a text draw and blits through `sd_canvas_blit_at`), crab
(pins sadish directly, 0.5.4 as of crab 0.8.10) and agnos's refagree GPU test
— pull `dist/sadish.cyr` via a `[deps.sadish]` git-tag entry (rekha and
dhancha also carry a `path = "../sadish"` dev override). The complete 2D vector
core — fill, stroke, gradient, affine transforms, clip, and analytic AA — is
live as of **v0.4.0**; styled strokes, gradient paint and exact 2-axis coverage
as of **v0.6.0**; dashes, focal gradients and premultiplied output as of **v0.7.0**;
bounded flattening and checked allocations as of **v0.7.1**; hook-safe strokes and
bounded fills as of **v0.7.2**; exact-size piece paths and a self-describing
polyline as of **v0.8.0**; inline path points as of **v0.9.0**.

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`. No external crate deps — sadish is a
  leaf. Resolved by `cyrius deps` into `lib/`.

All deps are pinned in `cyrius.cyml`; the toolchain pin is
`cyrius = "6.6.4"`.

## Quick Start

```bash
cyrius deps                                          # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/sadish-smoke    # link-check
./build/sadish-smoke                                  # prints the banner

# RUN tests (each self-checks and exits non-zero on failure)
for t in geom flatten fill blit rotate gradient grow stroke clip aa draw present blend alloc \
         stride stroke_style paint area integration grow_edges paint_focal premul \
         dash clip_pitch paint_premul flatten_bound oom stroke_oom path_cap \
         present_open inline_points; do
  cyrius build "programs/${t}_test.cyr" "build/${t}_test" && "./build/${t}_test"
done
```

## License

GPL-3.0-only.
