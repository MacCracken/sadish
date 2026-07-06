# sadish

Version: 0.1.0

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
    `SdMatrix` (2×3 affine a,b,c,d,e,f). `sd_point_new`/`_x`/`_y`;
    `sd_matrix_identity`/`_translate`/`_scale`/`_mul`/`_apply`. Identity,
    translate, scale, compose, and point-transform are **live** (exact
    fixed-point). Rotate / skew are TODO(v0.2).
  - **`src/path.cyr`** — `SdPath` (growable verb + point streams).
    `sd_path_new`/`_moveto`/`_lineto`/`_quadto`/`_cubicto`/`_close`, all
    live. `sd_path_flatten(path, tol) -> SdPolyline` is a **stub** that
    copies moveto/lineto anchors 1:1 and keeps curve end points without
    subdivision. Adaptive Bézier subdivision (de Casteljau) is TODO(v0.2).
  - **`src/raster.cyr`** — `SdCanvas` (width, height, owned width×height
    coverage buffer). `sd_canvas_new`/`_clear`/`_coverage_at` are live;
    `sd_canvas_fill_path(cv, path, rule)` flattens + validates then
    returns `SADISH_OK` (no coverage written yet); `sd_canvas_blit` is a
    validated no-op. Scanline coverage accumulation + AA and the src-over
    / gradient blit are TODO(v0.2).
  - **`src/error.cyr`** — `SadishErr` (16-byte record): `SADISH_OK` /
    `_ERR_OOM` / `_ERR_EMPTY_PATH` / `_ERR_BOUNDS` / `_ERR_UNSUPPORTED` /
    `_ERR_OTHER`, with `sadish_err_new`/`_err`/`_err_code`/`_err_detail`/
    `_err_name`.
  - **`programs/smoke.cyr`** — link-check entry proving the include chain
    (stdlib + domain modules) parses and links; writes a banner, exits 0.
- **v0.2.0 — the rasterizer.** Adaptive Bézier flattening, the scanline
  signed-area coverage pass (even-odd + nonzero) with anti-aliased 0..255
  coverage, and the src-over solid + gradient blit (with the AGNOS
  `blit#39` fast path). This is where sadish stops being a scaffold.
- **Later (tracked, not silently dropped):** matrix rotate/skew, clip
  paths / clip stack, stroking (a stroke → fill-path expansion), and dash
  patterns.

## Place in the stack

sadish is a **leaf** — no upstream sadish dependencies (only the Cyrius
stdlib). It is the bottom of the vector-graphics tower:

```
  dhancha (UI toolkit) ─┐
  SVG / canvas / compositing ─┤
  rekha (font outlines) ──────┴─▶ sadish  (this repo: paths + rasterizer + coverage)
                                     │
                                     ▼
                          coverage buffer ─▶ blit#39 (AGNOS) / host framebuffer
```

Everything above consumes sadish's path API and coverage buffer; none of
them re-implement rasterization.

## Consumers

- **rekha** — font outline rendering (glyph contours → sadish paths →
  coverage).
- **dhancha** — the AGNOS UI toolkit (widgets draw through sadish).
- A canvas / drawing API, compositing, and SVG layers (planned) sit on the
  same core.

No live consumer depends on sadish yet (it is a fresh scaffold);
downstream repos pull `dist/sadish.cyr` via a `[deps.sadish]` git-tag
entry once the v0.2 rasterizer lands.

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`. No external crate deps — sadish is a
  leaf. Resolved by `cyrius deps` into `lib/`.

All deps are pinned in `cyrius.cyml`; the toolchain pin is
`cyrius = "6.4.7"`.

## Quick Start

```bash
cyrius deps                                        # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/sadish-smoke  # link-check
./build/sadish-smoke                                # prints the banner
```

## License

GPL-3.0-only.
