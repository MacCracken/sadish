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
