# Clip masks are WRITTEN packed (`y * w + x`) but READ by the canvas stride (`py * stride + px`)

**Status:** 🟡 **OPEN — the FILED DEFECT IS FIXED (0.7.0); the gate claim this header used to make was
not true.** Three coverage loads are still unpinned; see the two ⚠/⛔ blocks below and the foot of the
file. Corrected at the 2026-09-15 re-audit against 0.7.1 (HEAD `f722e8c`).
**FIXED in 0.7.0** — the contract below was adopted verbatim: **a clip mask is PACKED
`w*h`, on a canvas of any stride**. All three readers now index `mask + py * w + px`
(`src/raster.cyr` `sd_fill_impl`, `src/coverage.cyr` `_sd_area_fill`, `src/stroke.cyr`
`_sd_sb_flush`), the contract is stated in raster.cyr's `SdCanvas` layout comment, in the clipping
section header and at each of the three readers, and the gate is the new suite
**`programs/clip_pitch_test.cyr`** (106 checks) — a hand-built 10x7 canvas header over a 16-byte-pitch
guarded buffer, which `sd_canvas_new` cannot produce.
⛔ **This line read "CLOSED in 0.6.1", and there is no 0.6.1.** The tags go 0.5.5 → 0.6.0 → 0.7.0 →
0.7.1 and CHANGELOG.md has no `## [0.6.1]`; `git log -S "py * w + px" -- src/raster.cyr` returns
exactly one commit, `536a3b4` (0.7.0), and the `### Fixed` entry sits inside the 0.7.0 section. The
same in-development label is baked into the SHIPPED comment at `src/raster.cyr:38` and about a dozen
other src comments, and `cyrius distlib` copies those verbatim into `dist/sadish.cyr` — so every
consumer of the bundle reads a version number that was never cut.
⚠ **The suite does NOT pin the whole coverage side**, which is what this header used to claim (groups
A, H, I, J, "so it cannot be satisfied by making everything packed instead"). RE-MEASURED on the 27
suites at 0.7.1: THREE coverage loads can each be pushed to the packed index with **all 27 suites
green** — `src/raster.cyr:953` in the public `sd_canvas_blit_gradient`, and `src/paint.cyr:1113` /
`:1302`, the `pm != 0` branches reached only through `sd_canvas_blit_paint_premul_at` /
`sd_canvas_blit_paint_premul`. Group J drives `blit_at`, `blit_paint_at` and `_sd_paint_blit_point`,
never `blit_gradient`, and only ever at `pm = 0`. All three loads are CORRECT in the tree; none is
gated, and that is exactly the condition the closing paragraph says it swept for.
**Fix landed:** 2026-09-15. **Byte-identical**, as predicted: every pre-existing suite passes with its
assertions unedited (**22** at the time — the tree carries **27** today, and the 0.7.0 CHANGELOG's own
block counts them a third way, "all 25 … the 19 pre-0.7.0 suites"; derive, do not quote) and agnos's
`refagree` prints *BYTE-IDENTICAL on all 200 paths*. ⚠ That `refagree` run was against **0.7.0's**
`dist/`. 0.7.1 rewrote `sd_fill_impl` around the untouched pixel loop (`_sd_fill_trunc` / `flat_rc`,
`ectx == 0` and `accrow == 0` OOM returns, null-point guards on LINETO/QUADTO/CUBICTO/CLOSE); there is
no 0.7.1 restatement, and `refagree` is an agnos-side oracle with no harness in this repo.
**Proof the gate bites** — RE-MEASURED at the 2026-09-15 re-audit, every count below reproduced to the
digit (each mutation applied alone, then reverted): reverting any one reader to
`py * stride + px` fails `clip_pitch_test` and NO other suite — `sd_fill_impl` 18 checks,
`_sd_area_fill` 6, `_sd_sb_flush` 7. Pushing the WRITER the other way
(`sd_canvas_clip_push_rect` storing at `y * stride + x`) fails it too, 23 checks — the suite pins the
contract from both sides. An index MIRRORED in x (`py * w + (w - 1 - px)`) fails it in 15. The 0.6.0 verifier's "clip mask read by `w`" mutation is therefore no
longer equivalent; it is now the code, and its inverse is caught.
⚠ **The clip rectangle is asymmetric on purpose.** The first draft used `[3,7)` on a 10-wide canvas,
which the mirror `x -> 9 - x` maps onto itself: `py * w + (w - 1 - px)` passed all of it. `[2,5)` in x
is disjoint from its mirror (`{2,3,4} -> {7,6,5}`), so half a mis-built index cannot hide.
⛔ **This block used to say "asymmetric in BOTH axes … disjoint from both of its mirrors", and the y
half is wrong.** On `h = 7` the mirror of `[2,6)` is `{2,3,4,5} -> {4,3,2,1}` = `[1,5)`, which OVERLAPS
the original in rows 2, 3 and 4. `clip_pitch_test.cyr:86` states it correctly ("`[2, 6)` on `h = 7`
mirrors to `[1, 5)`, which differs in 2 rows"); this file overstated its own suite. The conclusion
survives the correction — RE-MEASURED, a y-mirrored reader (`(h - 1 - py) * w + px`) fails
`clip_pitch_test` in 8 checks, plus `clip_test` 1 and `area_test` 1 — but the stated reason did not.
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
⚠ Re-checked 2026-09-15: the PIXEL LOOP is still untouched — diffed against `41b365b` (0.6.0), the
only change inside it is `py * stride + px` → `py * w + px` plus a trailing comment. The ENCLOSING
`sd_fill_impl` was rewritten in 0.7.1, and the `refagree` run recorded above was against 0.7.0's
`dist/`.

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
only the window). MEASURED: reverting to the flat run fails 2 of its checks (#92, #93) and NO other
suite — before the group, the flat run could be changed to `h * stride` and every suite still passed
(23 at the time; 27 today).
⚠ Every other group in that suite still zeroes its own window by hand in `cp_rig`, so a regression in
`sd_canvas_clear` cannot silently seed a later group.

Four more coverage loads were correct but **untested** for the same reason — nothing could build a
canvas that told them apart: `sd_fill_impl`'s `combine != 0` read-back (the one
`sd_canvas_fill_union` takes), `sd_canvas_blit_at`, `sd_canvas_blit_paint_at`'s gradient loop and
`_sd_paint_blit_point`'s per-pixel loop. Pushing any of them to the packed index survived every suite
(23 at the time; 27 today). `clip_pitch_test` groups H and J close **those four** — RE-MEASURED at
0.7.1, pushing them packed now fails 7, 1, 1 and 1 checks, `clip_pitch_test` only. No source change
was needed.

## ⚠ Still open — three coverage loads the sweep missed

The paragraph above reads as a closed class, and CHANGELOG.md:277-279 tallies it as one ("a clip
reviewer found six coverage loads no canvas could reach … `clip_pitch_test` groups H and J close
them"). It is not closed. RE-MEASURED 2026-09-15 on 0.7.1, pushing each of these to the packed index
ALONE leaves **all 27 suites green**:

| site | why the gate cannot see it |
|---|---|
| `src/raster.cyr:953` `sd_canvas_blit_gradient` — `load8(covbuf + y * covstride + x)` | predates the fix; group J never calls `sd_canvas_blit_gradient` |
| `src/paint.cyr:1113` `_sd_paint_blit_run`, `pm != 0` branch | reached only via `sd_canvas_blit_paint_premul_at` / `_premul`; group J drives `pm = 0` only |
| `src/paint.cyr:1302` `_sd_paint_blit_point`, `pm != 0` branch | same |

⚠ `src/premul.cyr:393` (`sd_canvas_blit_premul_at`) is in the same family and IS gated — by
`premul_test`'s own stride-12 hand-built canvas (`programs/premul_test.cyr:1252`), 2 checks. So the
pattern is reachable by a test; these three simply have no caller in one.

**What would close this filing:** a group K in `programs/clip_pitch_test.cyr` that (a) blits the
16-pitch canvas and its packed twin through `sd_canvas_blit_gradient` onto two surfaces and compares,
and (b) does the same through `sd_canvas_blit_paint_premul_at` for a linear and a focal gradient.
Three oracle comparisons; **no source change** — all three loads are already correct. Then correct the
0.7.0 label in `src/raster.cyr:38` and the sibling src comments, and re-run `cyrius distlib`.
