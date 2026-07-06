# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

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
