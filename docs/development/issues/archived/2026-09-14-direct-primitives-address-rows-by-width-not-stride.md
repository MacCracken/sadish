# Every direct primitive addresses rows by `width * 4`, so a WRAPPED surface (stride != width*4) shears

> 📁 **ARCHIVED — the Status line below is current; everything after it is AS FILED.** In-body
> `file:line` citations, suite counts and any "this filing does not archive" line were true when
> written and are **not maintained**: re-pointing them would edit the measurement, and the
> measurement is the point. Where the body and the Status line disagree, the Status line wins.
> ⇒ For what is actually left, read [`../../roadmap.md`](../../roadmap.md).

**Status:** 🟢 **CLOSED in 0.6.0** — every site in the table below (and `sd_surface_write_ppm`, which the
table missed: it read `w*h` pixels as one flat run) now loads `sd_surface_stride` once per call.
Gate: `programs/stride_test.cyr`, a hand-built wrapped header with sentinel padding and guard rows
driven through every primitive, blit, the gradient, the PPM writer and the presenter copy — 58 of its
103 checks fail against the 0.5.5 sources. Packed surfaces are byte-identical (all 14 pre-existing
suites unchanged). ⚠ `sd_put(px, w, h, x, y, color)` keeps its signature and stays the PACKED form
(a raw pointer carries no stride); no PRODUCTION call site is left in sadish or its consumers.
**Archived:** 2026-09-15, re-audited against 0.7.1 (HEAD `f722e8c`). RE-MEASURED at that audit:
today's `programs/stride_test.cyr` built against `git archive 0.5.5 src` exits **58** of 103 — the
figure above, to the digit — and ten single-site reversions to the 0.5.5 width-as-pitch form, applied
one at a time, each fail it (5, 8, 13, 14, 10, 12, 9, 11, 3 and 4 checks). No named site is un-gated.
⚠ The Status line's `sd_put` clause is narrower than it reads: `programs/stride_test.cyr` group K
(:475-476) calls `sd_put` deliberately, as the only thing left holding its arithmetic. The test file
says so; this line did not.
⚠ **Two consumer repos still vendor the PRE-FIX bundle** and therefore still carry the sheared
`sd_hline` / `sd_vline` / `sd_put`: `puka/lib/sadish.cyr` (0.5.4) and `crab/lib/sadish.cyr` (0.5.5).
Neither was checked for a wrapped-surface construction. The fix reached this repo, not those copies.
**Was:** 🟡 OPEN — pre-existing, made visible by 0.5.5. MEASURED, not read.
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

⚠ **The `file:line` column is AS FILED, against 0.5.5.** Every one of the seven was exact at that tag
(re-verified at the 0.7.1 audit with `git show 0.5.5:src/<file>`); three releases moved all seven. The
same sites at 0.7.1: `sd_put` `src/surface.cyr:144-146`, `sd_surface_pixel_at` `:150` (pitch at
`:158`), `sd_hline` `src/draw.cyr:63` (pitch `:74`), `sd_vline` `:88` (pitch `:94`, `:102`),
`sd_blend_hline` `:143` (pitch `:162`), `sd_canvas_blit_gradient` `src/raster.cyr:922` (pitch `:932`
— 291 lines of drift, the worst of the set), and the presenter's row copy inside `sd_present_blit`
`src/present.cyr:217` (stride loaded at `:229`, used at `:244`). The FUNCTION NAMES are the durable
reference here; the line numbers were never going to be.

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

⚠ **Two deviations between this plan and what 0.6.0 shipped.** Recorded at the 0.7.1 archive audit,
because a reader checking the plan bullet by bullet will not find either in the code:

- The plan's ONE shared store is `sd_put`, and `sd_put` was NOT made stride-aware — a raw pointer
  carries no stride. 0.6.0 added a private row-pointer store instead, `_sd_put_row(row, w, h, x, y,
  color)` (`src/surface.cyr:127` at 0.7.1, unmoved at 0.7.2), and every primitive hands it
  `px + y * stride`; `sd_put` (`src/surface.cyr:144-146`, likewise) still computes
  `px + y * (w * 4)` and is now packed by contract. The
  INVARIANT this bullet served is met by a different route, and the Status line says so — but no code
  satisfies the bullet as written.
- The list is short by one site: it names `sd_put`, `sd_surface_pixel_at`, "the three row-based fast
  paths" and the presenter copy, and omits `sd_vline`, which this filing's OWN table lists at
  `src/draw.cyr:89`. 0.6.0 covered it anyway (`sd_vline`'s stride load and its use,
  `src/draw.cyr:94`, `:102` at 0.7.1, unmoved at 0.7.2) and `stride_test` group E
  pins it — MEASURED, reverting that one site fails 10 checks. A gap in the plan text, not the
  outcome.

## Related

- dhancha `src/canvas.cyr` `dh_surface_wrap` — the consumer that constructs such surfaces, and whose
  own pixel loops (`dh_canvas_blit_rgb24`, the kashi bitmap text) already honour the stride.
- `dhancha/docs/development/issues/2026-09-13-scalable-text-allocates-per-call-outside-the-frame-arena.md`
  — the filing whose fix introduced `sd_canvas_blit_at`. ⚠ That path is relative to DHANCHA's repo
  root, not this one; re-verified 2026-09-15 that it still resolves there (dhancha has no `archived/`).
