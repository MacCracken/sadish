# sadish

Version: 0.4.1

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
- **v0.5.0 — next:** miter/bevel joins + butt/square caps, radial + multi-stop
  gradients, full 2-axis signed-area coverage, and the `rekha` / `dhancha`
  consumers coming online.

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
`dist/sadish.cyr` via a `[deps.sadish]` git-tag entry. The complete 2D vector
core — fill, stroke, gradient, affine transforms, clip, and analytic AA — is
live as of **v0.4.0**.

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`. No external crate deps — sadish is a
  leaf. Resolved by `cyrius deps` into `lib/`.

All deps are pinned in `cyrius.cyml`; the toolchain pin is
`cyrius = "6.4.7"`.

## Quick Start

```bash
cyrius deps                                          # resolve stdlib into lib/
cyrius build programs/smoke.cyr build/sadish-smoke    # link-check
./build/sadish-smoke                                  # prints the banner

# RUN tests (each self-checks and exits non-zero on failure)
for t in geom flatten fill blit rotate gradient grow stroke clip aa draw present; do
  cyrius build "programs/${t}_test.cyr" "build/${t}_test" && "./build/${t}_test"
done
```

## License

GPL-3.0-only.
