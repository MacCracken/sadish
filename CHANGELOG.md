# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

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
