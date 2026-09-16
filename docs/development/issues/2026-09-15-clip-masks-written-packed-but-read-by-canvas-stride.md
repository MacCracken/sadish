# Clip masks are WRITTEN packed (`y * w + x`) but READ by the canvas stride (`py * stride + px`)

**Status:** 🟢 **CLOSED** in 0.6.1 — the contract below was adopted verbatim: **a clip mask is PACKED
`w*h`, on a canvas of any stride**. All three readers now index `mask + py * w + px`
(`src/raster.cyr` `sd_fill_impl`, `src/coverage.cyr` `_sd_area_fill`, `src/stroke.cyr`
`_sd_sb_flush`), the contract is stated in raster.cyr's `SdCanvas` layout comment, in the clipping
section header and at each of the three readers, and the gate is the new suite
**`programs/clip_pitch_test.cyr`** (106 checks) — a hand-built 10x7 canvas header over a 16-byte-pitch
guarded buffer, which `sd_canvas_new` cannot produce. The suite pins the COVERAGE side at the stride
in the same breath (groups A, H, I, J), so it cannot be satisfied by making everything packed instead.
**Closed by:** 2026-09-15. **Byte-identical**, as predicted: the 22 pre-existing suites pass with
their assertions unedited and agnos's `refagree` prints *BYTE-IDENTICAL on all 200 paths*.
**Proof the gate bites** (each mutation applied alone, then reverted): reverting any one reader to
`py * stride + px` fails `clip_pitch_test` and NO other suite — `sd_fill_impl` 18 checks,
`_sd_area_fill` 6, `_sd_sb_flush` 7. Pushing the WRITER the other way
(`sd_canvas_clip_push_rect` storing at `y * stride + x`) fails it too, 23 checks — the suite pins the
contract from both sides. An index MIRRORED in x (`py * w + (w - 1 - px)`) fails it in 15. The 0.6.0 verifier's "clip mask read by `w`" mutation is therefore no
longer equivalent; it is now the code, and its inverse is caught.
⚠ **The clip rectangle is asymmetric in BOTH axes on purpose.** The first draft used `[3,7)` on a
10-wide canvas, which the mirror `x -> 9 - x` maps onto itself: `py * w + (w - 1 - px)` passed all of
it. `[2,5) x [2,6)` is disjoint from both of its mirrors, so half a mis-built index cannot hide.
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

## What closed it

Decide the mask's pitch ONCE. The mask is sadish-owned and never wrapped, so the simplest contract is
"masks are always packed `w*h`": read `mask + py * w + px` in the three readers (byte-identical today,
since `stride == w`) and state it in the SdCanvas layout comment. The gate is a hand-built canvas
header with `stride != width` over a guarded coverage buffer, a rect clip pushed onto it, and a fill /
AREA fill / styled stroke asserting the clipped region lands where the mask says.

⛔ Do NOT change the SUBSCANLINE loop's arithmetic in any other way — agnos's refagree oracle is gated
on its bytes (an unclipped fill never reads the mask, so the fix cannot move them).
Nothing else in the loop was touched; `refagree` re-ran green against the fixed `dist/`.

## Also fixed: `sd_canvas_clear` zeroed `w*h` from the coverage pointer

Found next door while building the gate, and fixed with it after review. `sd_canvas_clear`
(`src/raster.cyr`) walked `i < w * h` storing 0 at `cov + i`, ignoring `SD_CANVAS_STRIDE_OFFSET` — the
COVERAGE side of the same confusion. On a `stride == width` canvas that flat run is exactly the
buffer; on a wrapped one it clears neither all of the window nor only the window. On
`clip_pitch_test`'s 10x7 canvas at a 16-byte pitch it would zero rows 0..3 entirely (the window AND
24 bytes of padding), reach only column 5 of row 4, and leave 24 window bytes of rows 4..6 dirty.
It now walks rows at the stride, `w` bytes a row — byte-identical on every canvas `sd_canvas_new`
makes, where the two walks are the same bytes in the same order.
**Gate:** `programs/clip_pitch_test.cyr` group I checks both halves (the whole window zeroed, and
only the window). MEASURED: reverting to the flat run fails 2 of its checks and NO other suite —
before the group, the flat run could be changed to `h * stride` and all 23 suites still passed.
⚠ Every other group in that suite still zeroes its own window by hand in `cp_rig`, so a regression in
`sd_canvas_clear` cannot silently seed a later group.

Four more coverage loads were correct but **untested** for the same reason — nothing could build a
canvas that told them apart: `sd_fill_impl`'s `combine != 0` read-back (the one
`sd_canvas_fill_union` takes), `sd_canvas_blit_at`, `sd_canvas_blit_paint_at`'s gradient loop and
`_sd_paint_blit_point`'s per-pixel loop. Pushing any of them to the packed index survived all 23
suites. `clip_pitch_test` groups H and J close all four; no source change was needed.
