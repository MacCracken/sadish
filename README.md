# sadish

Version: 0.9.1

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

The complete 2D vector core is live: paths, curve flattening, analytic anti-aliased
fill (even-odd + nonzero), styled and dashed stroking, linear/radial/focal gradient
paint, premultiplied output, affine transforms, clipping, and an opt-in exact
2-axis coverage engine — on an allocation seam a consumer can point at its own
arena.

Current release **0.9.1**. What shipped in each version is in
[`CHANGELOG.md`](./CHANGELOG.md); what is left, and what 1.0 still needs, is in
[`docs/development/roadmap.md`](./docs/development/roadmap.md).

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
polyline as of **v0.8.0**; inline path points as of **v0.9.0**; portable syscall
constants and an aarch64 + AGNOS cross-build gate as of **v0.9.1**.

## Dependencies

- **Cyrius stdlib** — `string`, `fmt`, `alloc`, `io`, `vec`, `str`,
  `syscalls`, `assert`, `bench`. No external crate deps — sadish is a
  leaf. Resolved by `cyrius deps` into `lib/`.

All deps are pinned in `cyrius.cyml`; the toolchain pin is
`cyrius = "6.6.6"`.

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
