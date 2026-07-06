# sadish

Version: 0.2.0

**sadish** (सदिश — *sa* "with" + *diś* "direction" = "having direction":
the modern Sanskrit/Hindi word for **vector**; antonym अदिश *adish* =
scalar) is a pure-Cyrius, low-level **2D vector graphics core** for
AGNOS — the sovereign foundation everything vector builds up from.

It turns resolution-independent geometry into pixels: path construction
(move / line / quadratic + cubic Bézier), curve flattening, scanline fill
(even-odd + nonzero winding) with anti-aliased coverage, affine
transforms, clipping, and solid + gradient fills → a pixel/coverage buffer
that blits via the kernel `blit#39` (or a host framebuffer under test).
No GPU, no C shim, no external binaries — everything-is-i64 integer
arithmetic (16.16 fixed-point) all the way down.

sadish is the **low-level lib you build up from**: `rekha` (font
outlines), a canvas/drawing API, compositing, SVG, and the `dhancha` UI
toolkit are all **consumers**, not re-implementations.

## Scope

- **v0.1.0 — scaffold.** A buildable, link-clean pure-Cyrius skeleton —
  **not** the rasterizer yet. The module shapes are real (types + public
  fn signatures compile and link), the algorithm bodies are stubs marked
  `# TODO(v0.x)`:
  - **`src/geom.cyr`** — `SdPoint` (x, y as 16.16 fixed-point) and
    `SdMatrix` (2×3 affine a,b,c,d,e,f). Identity, translate, scale,
    compose, and point-transform are **live** (exact fixed-point). Rotate
    / skew are TODO(v0.3).
  - **`src/path.cyr`** — `SdPath` (growable verb + point streams).
    `sd_path_new`/`_moveto`/`_lineto`/`_quadto`/`_cubicto`/`_close`, all
    live. `sd_path_flatten(path, tol) -> SdPolyline` is a **stub** that
    copies moveto/lineto anchors without subdivision. Adaptive Bézier
    subdivision (de Casteljau) is TODO(v0.3).
  - **`src/raster.cyr`** — `SdCanvas` (owned width×height coverage
    buffer). `sd_canvas_new`/`_clear`/`_coverage_at` are live;
    `sd_canvas_fill_path` flattens + validates then returns `SADISH_OK`
    (no coverage written yet); `sd_canvas_blit` is a validated no-op.
    Scanline coverage accumulation + AA and the src-over / gradient blit
    are TODO(v0.3).
  - **`src/error.cyr`** — `SadishErr` (16-byte record) + the `SADISH_*`
    codes and `sadish_err_*` helpers.
  - **`programs/smoke.cyr`** — link-check entry.
- **v0.2.0 — surface + direct primitives + present (shipped).** The
  "easy 80%" lifted from the ecosystem's framebuffer games (see
  `docs/development/prior-art.md`), CI-gated and RUN-tested:
  - **`src/surface.cyr`** — `SdSurface`, a 32bpp BGRA pixel buffer (packed
    `0x00RRGGBB`, top-left y-down), `sd_rgb` + channel extractors, pixel
    readback.
  - **`src/draw.cyr`** — the opaque integer-pixel primitives: `sd_plot`,
    `sd_line` (integer Bresenham), `sd_hline`, `sd_vline`, `sd_rect`,
    `sd_fill_rect`, `sd_clear`.
  - **`src/present.cyr`** — `sd_surface_write_ppm` (P6 PPM, headless/CI)
    and the Linux `/dev/fb0` `SdPresenter` (probe → integer-scale →
    center-letterbox → pitch-honoring blit; 32bpp + RGB565).
  - Tests: `programs/draw_test.cyr` (pixel-readback) and
    `programs/present_test.cyr` (PPM write + readback).
- **v0.3.0 — the rasterizer.** Adaptive Bézier flattening, the scanline
  signed-area coverage pass (even-odd + nonzero) with anti-aliased 0..255
  coverage, and the src-over solid + gradient blit compositing coverage
  onto an `SdSurface` (with the AGNOS `blit#39` fast path). This is where
  the vector core (paths → coverage) comes alive.
- **Later (tracked, not silently dropped):** matrix rotate/skew, clip
  paths / clip stack, stroking (stroke → fill-path expansion), and dash
  patterns.

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

No live consumer depends on sadish yet; downstream repos pull
`dist/sadish.cyr` via a `[deps.sadish]` git-tag entry. The direct
primitives + present backend are usable at **v0.2.0**; the vector
rasterizer lands at **v0.3.0**.

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`. No external crate deps — sadish is a
  leaf. Resolved by `cyrius deps` into `lib/`.

All deps are pinned in `cyrius.cyml`; the toolchain pin is
`cyrius = "6.4.7"`.

## Quick Start

```bash
cyrius deps                                         # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/sadish-smoke   # link-check
./build/sadish-smoke                                 # prints the banner

cyrius build programs/draw_test.cyr build/draw_test  # primitives RUN test
./build/draw_test                                    # -> draw_test: PASS
cyrius build programs/present_test.cyr build/present_test
./build/present_test                                 # -> present_test: PASS
```

## License

GPL-3.0-only.
