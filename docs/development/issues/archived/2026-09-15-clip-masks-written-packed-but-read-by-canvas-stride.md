# Clip masks are WRITTEN packed (`y * w + x`) but READ by the canvas stride (`py * stride + px`)

**Status:** 🟢 **CLOSED — the FILED DEFECT was fixed in 0.7.0; the GATE GAP this header used to hide
was closed in 0.7.2.** The three unpinned coverage loads are now gated by `programs/clip_pitch_test.cyr`
group K (checks 107-122, three oracle comparisons against the packed twin); **no source change was
needed** — all three were already correct, which is what the group proves. The phantom `0.6.1` labels
this file also asked for are already gone from the tree: `grep -rn "0\.6\.1"` over `src/*.cyr`,
`programs/*.cyr` and `dist/sadish.cyr` returns **0 hits** at 0.7.2 (CHANGELOG.md:132 keeps one, as the
history of the correction). Ready to archive; the `Closes` line in CHANGELOG.md and the `git mv` into
`archived/` are the maintainer's.
Was 🟡 OPEN at the 2026-09-15 re-audit against 0.7.1 (HEAD `f722e8c`), which found the gap.
**FIXED in 0.7.0** — the contract below was adopted verbatim: **a clip mask is PACKED
`w*h`, on a canvas of any stride**. All three readers now index `mask + py * w + px`
(`src/raster.cyr` `sd_fill_impl`, `src/coverage.cyr` `_sd_area_fill`, `src/stroke.cyr`
`_sd_sb_flush`), the contract is stated in raster.cyr's `SdCanvas` layout comment, in the clipping
section header and at each of the three readers, and the gate is the new suite
**`programs/clip_pitch_test.cyr`** (106 checks then; **122** since group K landed in 0.7.2) — a
hand-built 10x7 canvas header over a 16-byte-pitch guarded buffer, which `sd_canvas_new` cannot
produce.
⛔ **This line read "CLOSED in 0.6.1", and there is no 0.6.1.** The tags go 0.5.5 → 0.6.0 → 0.7.0 →
0.7.1 and CHANGELOG.md has no `## [0.6.1]`; `git log -S "py * w + px" -- src/raster.cyr` returns
exactly one commit, `536a3b4` (0.7.0), and the `### Fixed` entry sits inside the 0.7.0 section. The
same in-development label WAS baked into the SHIPPED comment in raster.cyr's `SdCanvas` layout note
and about a dozen other src comments, and `cyrius distlib` copies those verbatim into
`dist/sadish.cyr` — so every consumer of the bundle read a version number that was never cut. ⭐ That
half is already repaired in the tree: RE-MEASURED at 0.7.2, `grep -rn "0\.6\.1"` over `src/*.cyr`,
`programs/*.cyr` and `dist/sadish.cyr` returns 0 hits, and the layout note now reads "since 0.7.0 the
three consumers read it packed too".
⚠ **Through 0.7.1 the suite did NOT pin the whole coverage side**, which is what this header used to
claim (groups A, H, I, J, "so it cannot be satisfied by making everything packed instead").
RE-MEASURED on the 27 suites at 0.7.1: THREE coverage loads could each be pushed to the packed index
with **all 27 suites green** — `sd_canvas_blit_gradient`'s own load (`src/raster.cyr`), and the
`pm != 0` coverage argument in `_sd_paint_blit_run` and in `_sd_paint_blit_point` (`src/paint.cyr`),
reached only through `sd_canvas_blit_paint_premul_at` / `sd_canvas_blit_paint_premul`. Group J drives
`blit_at`, `blit_paint_at` and `_sd_paint_blit_point`, never `blit_gradient`, and only ever at
`pm = 0`. All three loads are CORRECT in the tree; none was gated, and that is exactly the condition
the closing paragraph said it had swept for. **Group K closed all three in 0.7.2** — see the foot of
the file for the re-measured mutation counts.
**Fix landed:** 2026-09-15. **Byte-identical**, as predicted: every pre-existing suite passes with its
assertions unedited (**22** at the time — the tree carries **27** today, and the 0.7.0 CHANGELOG's own
block counts them a third way, "all 25 … the 19 pre-0.7.0 suites"; derive, do not quote) and agnos's
`refagree` prints *BYTE-IDENTICAL on all 200 paths*. ⚠ That `refagree` run was against **0.7.0's**
`dist/`. 0.7.1 rewrote `sd_fill_impl` around the untouched pixel loop (`_sd_fill_trunc` / `flat_rc`,
`ectx == 0` and `accrow == 0` OOM returns, null-point guards on LINETO/QUADTO/CUBICTO/CLOSE); there is
no 0.7.1 restatement, and `refagree` is an agnos-side oracle with no harness in this repo. ⭐ There is
a **0.7.2** one: the oracle was re-run against this tree's `dist/` while group K landed (and while
`sd_fill_impl` took the flatten scope of the sibling filing) and printed *BYTE-IDENTICAL on all 200
paths*, worst delta 0.
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

## CLOSED in 0.7.2 — the three coverage loads the sweep missed

The paragraph above read as a closed class, and CHANGELOG.md:277-279 tallies it as one ("a clip
reviewer found six coverage loads no canvas could reach … `clip_pitch_test` groups H and J close
them"). It was not closed. RE-MEASURED 2026-09-15 on 0.7.1, pushing each of these to the packed index
ALONE left **all 27 suites green**:

| site | why the gate could not see it | what pins it now |
|---|---|---|
| `sd_canvas_blit_gradient`'s own coverage load (`src/raster.cyr`) | predates the fix; group J never calls `sd_canvas_blit_gradient` | group K check **110** |
| `_sd_paint_blit_run`, the `pm != 0` coverage argument (`src/paint.cyr`) | reached only via `sd_canvas_blit_paint_premul_at` / `_premul`; group J drives `pm = 0` only | group K check **114** |
| `_sd_paint_blit_point`, the same argument (`src/paint.cyr`) | same | group K check **118** |

⚠ `sd_canvas_blit_premul_at` (`src/premul.cyr`) is in the same family and was already gated — by
`premul_test`'s own hand-built canvas over a 12-byte pitch (`programs/premul_test.cyr:1228`, built as
a "(7 + 5)-byte pitch"). RE-MEASURED 2026-09-15, pushing its `crow` to `covbuf + y * cw + x0` alone:
`premul_test` **115 and 117** fail (an earlier draft of this line said 115 and 116 — wrong, and in a
file whose argument is that measured numbers reproduce to the digit). So the pattern was reachable by
a test; these three simply had no caller in one.

**What closed it:** `programs/clip_pitch_test.cyr` **group K** (checks 107-122) — the same 10x7
canvas over a 16-byte-pitch guarded buffer the rest of the file uses, carrying a styled stroke's
coverage, blitted onto a surface and compared byte for byte against the packed twin: once through
`sd_canvas_blit_gradient`, once through `sd_canvas_blit_paint_premul_at` with a LINEAR gradient and
once with a FOCAL one (the two premultiplied comparisons read **all four bytes** — the alpha byte is
what those blits exist for, and `sd_surface_pixel_at` drops it). **No source change: all three loads
were already correct**, which is the finding.

**Proof the gate bites** — MEASURED 2026-09-15 on the 0.7.2 tree, each mutation applied ALONE and then
reverted, full 27-suite sweep each time:

| mutation | result |
|---|---|
| `sd_canvas_blit_gradient` load → `y * cw + x` | `clip_pitch_test` #110, **and no other check in any suite** |
| `_sd_paint_blit_run`'s `pm != 0` argument → `y * cw + x` | `clip_pitch_test` #114, and nothing else |
| `_sd_paint_blit_point`'s `pm != 0` argument → `y * cw + x` | `clip_pitch_test` #118, and nothing else |

⚠ Each bites in exactly ONE check because each is one oracle comparison, as groups H and J are
(97 / 100 / 103). The five anti-vacuity checks beside them (111, 115, 119, 120, 121) are what stop
that comparison from being two blank surfaces: the pictures must carry ink, the premultiplied pair
must carry a PARTIAL alpha byte (only a composited alpha can leave one — the straight blits force
255), and the three pictures must differ from each other.
⚠ Row 0 of this canvas maps onto itself under the packed index (`0..9 < 16`), so a comparison confined
to it would pass. Row 1 reads bytes 10..19: six of them are row 0's padding, the sentinel 90, a
coverage no fill writes. Every offset a packed reader reaches stays inside the buffer (max
`6 * 10 + 9 = 69 < 112`), so the failure owes nothing to the heap.

**The `0.6.1` label this file also asked for is already gone.** RE-MEASURED at 0.7.2:
`grep -rn "0\.6\.1"` over `src/*.cyr`, `programs/*.cyr` and `dist/sadish.cyr` returns 0 hits;
`src/raster.cyr`'s SdCanvas layout comment now reads "since 0.7.0 the three consumers read it packed
too". CHANGELOG.md:132 keeps the only remaining mention, which is the record of the correction.
