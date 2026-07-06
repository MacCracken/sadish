# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] - unreleased

### Added
- Repo scaffolded: a buildable, link-clean pure-Cyrius skeleton for the
  low-level 2D vector graphics core — `SdPoint`/`SdMatrix` (16.16
  fixed-point geometry, live), `SdPath` (verb + point construction, live)
  with a `sd_path_flatten` stub, `SdCanvas` (coverage buffer) with
  `sd_canvas_fill_path`/`_blit` stubs, the `SadishErr` model, and a
  `programs/smoke.cyr` link-check. The scanline coverage rasterizer +
  adaptive Bézier flattening land in v0.2.0.
- **Pixel surface + direct primitives** (`src/surface.cyr`, `src/draw.cyr`) —
  `SdSurface`, a 32bpp BGRA pixel buffer (packed `0x00RRGGBB` colors, top-left
  y-down, `off=(y*w+x)*4`), plus the opaque integer-pixel primitives
  `sd_plot` / `sd_line` (integer Bresenham) / `sd_hline` / `sd_vline` /
  `sd_rect` / `sd_fill_rect` / `sd_clear`, and `sd_rgb` + channel extractors.
  These are the "easy 80%" **lifted, not reinvented** from the ecosystem's
  framebuffer games (encom-hits `draw.cyr` Bresenham; the BGRA/present
  convention shared with cyrius-bb/-polyomino/-doom) — see
  `docs/development/prior-art.md`. Verified by `programs/draw_test.cyr`, a
  headless pixel-readback RUN test (exact-diagonal Bresenham, span clamping,
  filled vs hollow rect, silent out-of-bounds clip). The anti-aliased coverage
  path (`SdCanvas`) stays separate; `sd_canvas_blit` will composite coverage
  onto an `SdSurface` in v0.2.0.
