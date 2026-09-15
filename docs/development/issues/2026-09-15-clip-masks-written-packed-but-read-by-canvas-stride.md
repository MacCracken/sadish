# Clip masks are WRITTEN packed (`y * w + x`) but READ by the canvas stride (`py * stride + px`)

**Status:** 🟡 **OPEN — latent, unobservable today. Found by reading**, confirmed by an equivalent mutation.
**Filed:** 2026-09-15, from the 0.6.0 stroke-item verification.
**Affects:** sadish **0.4.0 → 0.6.0** (the clip stack has always been this way).
**Severity:** **None today** — every `SdCanvas` is made by `sd_canvas_new`, which stores `stride == width`,
and there is no canvas-wrap API. **Real the day a canvas can wrap a wider coverage buffer** (the
coverage-side twin of dhancha's `dh_surface_wrap`).

## What was found

The canvas carries `SD_CANVAS_STRIDE_OFFSET`, and every coverage reader/writer that addresses the
COVERAGE buffer honours it. The clip MASK, however, is a separate `w*h` block, and its producers and
consumers disagree about its pitch:

| role | site | expression |
|---|---|---|
| write | `src/raster.cyr` `sd_canvas_clip_push_rect` | `shape + y * w + x` |
| write | `src/raster.cyr` `sd_clip_push_mask` (intersection) | `shape + i`, `i < w * h` |
| write | `src/raster.cyr` `sd_canvas_clip_push_path` | the temp canvas's coverage — stride of the TEMP canvas |
| read  | `src/raster.cyr` `sd_fill_impl` (SUBSCANLINE loop) | `mask + py * stride + px` |
| read  | `src/coverage.cyr` `_sd_area_fill` | `mask + py * stride + px` |
| read  | `src/stroke.cyr` `_sd_sb_flush` | `mask + py * stride + px` |

⚠ On a canvas whose stride differs from its width, every clipped fill, AREA fill and styled stroke
would read the mask one row of shear per row. The 0.6.0 stroke verifier's mutation "clip mask read
by `w`" survived as EQUIVALENT for exactly this reason: no canvas can tell the two apart.

## What would close it

Decide the mask's pitch ONCE. The mask is sadish-owned and never wrapped, so the simplest contract is
"masks are always packed `w*h`": read `mask + py * w + px` in the three readers (byte-identical today,
since `stride == w`) and state it in the SdCanvas layout comment. The gate is a hand-built canvas
header with `stride != width` over a guarded coverage buffer, a rect clip pushed onto it, and a fill /
AREA fill / styled stroke asserting the clipped region lands where the mask says.

⛔ Do NOT change the SUBSCANLINE loop's arithmetic in any other way — agnos's refagree oracle is gated
on its bytes (an unclipped fill never reads the mask, so the fix cannot move them).
