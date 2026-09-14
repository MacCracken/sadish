# Every direct primitive addresses rows by `width * 4`, so a WRAPPED surface (stride != width*4) shears

**Status:** 🟡 **OPEN — pre-existing, made visible by 0.5.5. MEASURED**, not read.
**Filed:** 2026-09-14, from the 0.5.5 review (the `sd_canvas_blit_at` work).
**Affects:** sadish **0.5.5** and every version since `SD_SURFACE_STRIDE_OFFSET` was added (the field
has been declared and written by `sd_surface_new` since v0.2, and READ by nothing in this repo until
`sd_canvas_blit_at`).
**Severity:** **Low today, high the day a consumer presents from a wrapped buffer.** Nothing in the
stack draws a background into a wrapped surface yet; dhancha's `dh_surface_wrap` exists precisely so
that it can.

## What was found

`sd_canvas_blit_at` (0.5.5) addresses rows by `sd_surface_stride(surface)`, and the review proved
scalable text lands straight on a `dh_surface_wrap`'d sub-rect (0 differing pixels against a packed
surface, 0 bytes written into the padding). The SAME wrapped tree with a `WINDOW` background —
`dh_fill_rect_clip` → `sd_fill_rect` — overwrote **4,000 padding pixels** (probe: a 200x60 header over
a 300-px-stride buffer with guard rows), because every other pixel writer still uses the surface
WIDTH as its row pitch:

| site | expression |
|---|---|
| `src/surface.cyr:117` `sd_put` (the shared store every primitive routes through) | `off = ((y * w) + x) * 4` |
| `src/surface.cyr:135` `sd_surface_pixel_at` (the shared read) | `off = ((y * w) + x) * 4` |
| `src/draw.cyr:62` `sd_hline` | `base = (y * w) * 4` |
| `src/draw.cyr:89` `sd_vline` | `off = ((y * w) + x) * 4` |
| `src/draw.cyr:149` `sd_blend_hline` (and so `sd_fill_rect_blend`) | `base = (y * w) * 4` |
| `src/raster.cyr:630` `sd_canvas_blit_gradient` | `off = (y * sw + x) * 4` |
| `src/present.cyr:161` the presenter's row copy | `src_row = srcpx + sy * (sw * 4)` |

⚠ Right for every packed surface sadish makes itself (`stride == width * 4` by construction in
`sd_surface_new`), one row of shear per row on a wrapped one — and `sd_canvas_blit_at` is now the
one writer that disagrees with the rest, which is a worse state than all of them agreeing on the
wrong answer: text lands straight and the rectangle under it does not.

## What would close it

Read `sd_surface_stride` in the ONE shared store (`sd_put`) and the ONE shared read
(`sd_surface_pixel_at`), then the three row-based fast paths (`sd_hline`, `sd_blend_hline`,
`sd_canvas_blit_gradient`) and the presenter copy. Packed surfaces are byte-identical either way
(`stride == w * 4`), so every existing suite must pass unchanged; the gate is a hand-built wrapped
header (the shape `programs/alloc_test.cyr` group E already builds for `blit_at`) through each
primitive, asserting row 1 lands at the pitch and the padding is untouched.

## Related

- dhancha `src/canvas.cyr` `dh_surface_wrap` — the consumer that constructs such surfaces, and whose
  own pixel loops (`dh_canvas_blit_rgb24`, the kashi bitmap text) already honour the stride.
- dhancha `docs/development/issues/2026-09-13-scalable-text-allocates-per-call-outside-the-frame-arena.md`
  — the filing whose fix introduced `sd_canvas_blit_at`.
