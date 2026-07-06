# sadish

Version: 0.3.0

**sadish** (सदिश — *sa* "with" + *diś* "direction" = "having direction":
the modern Sanskrit/Hindi word for **vector**; antonym अदिश *adish* =
scalar) is a pure-Cyrius, low-level **2D vector graphics core** for
AGNOS — the sovereign foundation everything vector builds up from.

It turns resolution-independent geometry into pixels: path construction
(move / line / quadratic + cubic Bézier), curve flattening, scanline fill
(even-odd + nonzero winding) with anti-aliased coverage, affine
transforms, and solid fills → a pixel/coverage buffer that blits via the
kernel `blit#39` (or a host framebuffer under test). No GPU, no C shim, no
external binaries — everything-is-i64 integer arithmetic (16.16
fixed-point) all the way down.

sadish is the **low-level lib you build up from**: `rekha` (font
outlines), a canvas/drawing API, compositing, SVG, and the `dhancha` UI
toolkit are all **consumers**, not re-implementations.

## Scope

- **v0.1.0 — scaffold.** Buildable, link-clean skeleton: real type shapes +
  public fn signatures (`SdPoint`/`SdMatrix`, `SdPath`, `SdCanvas`,
  `SadishErr`), algorithm bodies stubbed. Superseded by the releases below.
- **v0.2.0 — surface + direct primitives + present (shipped).** The "easy
  80%" lifted from the ecosystem's framebuffer games (see
  `docs/development/prior-art.md`):
  - **`src/surface.cyr`** — `SdSurface`, a 32bpp BGRA pixel buffer, `sd_rgb`
    + channel extractors, pixel readback.
  - **`src/draw.cyr`** — opaque integer-pixel primitives: `sd_plot`,
    `sd_line` (integer Bresenham), `sd_hline`, `sd_vline`, `sd_rect`,
    `sd_fill_rect`, `sd_clear`.
  - **`src/present.cyr`** — `sd_surface_write_ppm` (P6 PPM, headless/CI) and
    the Linux `/dev/fb0` `SdPresenter` (probe → integer-scale →
    center-letterbox → pitch-honoring blit; 32bpp + RGB565).
- **v0.3.0 — the rasterizer (shipped).** The vector core comes alive —
  paths → coverage → pixels, RUN-tested end to end:
  - **Signed fixed-point** (`sd_asr` / `sd_fixed_mul` / `sd_abs`) — a
    sign-preserving right shift (Cyrius `>>` is logical), so negatives
    (mirrors, rotations, below-origin coords) stay correct.
  - **`sd_path_flatten`** — adaptive de Casteljau subdivision (quad + cubic),
    flatness by second-difference vs `tol`.
  - **`sd_canvas_fill_path`** — closed-edge list from the flattened path, then
    supersampled (4×4) anti-aliased scanline coverage via +x winding rays;
    nonzero + even-odd.
  - **`sd_canvas_blit`** — composite coverage × a solid color, src-over, onto
    an `SdSurface`.
  - **`sd_matrix_rotate`** (`sd_cos`/`sd_sin`) — CORDIC fixed-point trig
    (shifts + adds, no float / no libm).
- **v0.4.0 — next:** analytic signed-area coverage (smoother AA, cheaper than
  supersampling), gradient paints, clip paths / clip stack, stroking
  (stroke → fill-path expansion), matrix skew, and a growable path/flatten
  backing (the fixed `SD_PATH_CAP` / `SD_FLATTEN_CAP` caps).

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
`dist/sadish.cyr` via a `[deps.sadish]` git-tag entry. The full
path → flatten → coverage → composite pipeline plus affine transforms are
live as of **v0.3.0**.

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
for t in geom flatten fill blit rotate draw present; do
  cyrius build "programs/${t}_test.cyr" "build/${t}_test" && "./build/${t}_test"
done
```

## License

GPL-3.0-only.
