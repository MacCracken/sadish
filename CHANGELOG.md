# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.4.0] - unreleased

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
