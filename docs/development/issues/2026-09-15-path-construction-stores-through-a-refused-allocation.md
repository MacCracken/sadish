# Path construction stores through a refused `sd_alloc` — a hook that returns 0 faults inside sadish

**Status:** 🟡 **OPEN — every site in the table below is FIXED (0.7.1); the filing's own goal is not
reached.** `sd_canvas_stroke_path` still SIGSEGVs under exactly the hook the 0.5.5 seam invites,
through two unchecked `sd_path_new()` calls sadish makes itself, and the seam contract this filing
asks to rewrite still reads the old way. See **Still open** at the foot. ⛔ This line read
"🟢 **CLOSED — FIXED in sadish 0.7.1**" until the 2026-09-15 re-audit against HEAD `f722e8c`.

**Fixed in 0.7.1**, in two halves that landed together: `src/path.cyr`
and `src/geom.cyr` (the path/point/flatten sites) and `src/raster.cyr`, `src/present.cyr`,
`src/error.cyr` (clip node, clip mask, PPM/presenter, error record). Every site in the table below
now checks its result and propagates — constructors return 0, path builders return `SADISH_ERR_OOM`
with the path unchanged, and the clip pushes leave the previous region intact and poppable. A
next-line scan of all **32** `sd_alloc(` result sites in `src/*.cyr` at 0.7.1 found ZERO unchecked
(re-run 2026-09-15). Gated by
`programs/oom_test.cyr` and `programs/flatten_bound_test.cyr`, which sweep a hook that refuses the
K-th allocation across every site in turn; thirteen mutations reinstating the 0.7.0 behaviour are
`SIGSEGV` rc 139 against **`oom_test`** — ⚠ that figure belongs to the raster/present/error half
alone, as the closing section says. RE-MEASURED 2026-09-15: exactly thirteen against `oom_test`, and
**seven MORE** SIGSEGV kills against `flatten_bound_test` (`sd_point_new`, the flatten out, the
flatten ctx, the polyline header, `sd_matrix_new`, `sd_matrix_invert`, `_sd_flat_widen`). The
top-line number under-counted its own gate.
⭐ **And the fill now REPORTS a starved flatten** rather than painting a wrong picture under
`SADISH_OK`: `sd_fill_impl` scopes the flatten's truncation verdict to the call and returns
`SADISH_ERR_OOM` when that fill lost points (MEASURED: a 4-cubic circle whose true ink is 312,280
coverage units paints 81,029 under a hook granting 5 allocations — 0.7.0 SIGSEGV'd there).
⚠ **Still the caller's job for STROKES**: the styled path has its own flush and the round stroker
discards `sd_canvas_fill_union`'s result, so `sd_flatten_truncated()` remains their only witness.
⛔ **That understates it, and the re-audit measured the difference.** It describes a REPORTING gap and
implies the stroke completes. It does not: a starved ROUND stroke **SIGSEGVs, rc 139**, through
`src/stroke.cyr:133` / `:145` (see **Still open**). There is no witness because there is no return.
The reporting gap is real for the STYLED stroke, which does complete — MEASURED, starved at 0, 1, 2
and 3 grants it returns `SADISH_OK` with 0 / 3,118 / 6,418 / 7,612 ink against a 7,756 baseline.
⚠ And the witness is sticky: nothing in a stroke clears `_sd_flat_trunc`, so it reads 1 on later
healthy strokes too. It cannot be trusted per call.
⚠ **One fault this filing did not name was found while closing it**, and is fixed here too: a path
whose FIRST verb is a drawing verb (no moveto) dereferenced a 0 `SdPoint` in every walk — fill,
both strokers, dash and `sd_path_flatten` — SIGSEGV on 0.7.0 and 0.6.0 alike. 0.7.1's clean
`SADISH_ERR_OOM` from a refused `sd_path_moveto` made it easy to build exactly that path (a caller
that logs and carries on), so the walks now skip drawing verbs until a moveto arrives, consuming
their points so the verb and point streams stay in step. Gated by `programs/integration_test.cyr`
group E.
Was 🟡 OPEN — MEASURED on sadish **0.6.0**.
**Filed:** 2026-09-15, by **rekha** (0.3.11 made every rekha allocation site check for 0; the remaining
fault is here).
**Affects:** `sd_path_new` (`src/path.cyr:48-49`), `sd_point_new` (`src/geom.cyr:67`), `sd_path_flatten`
(`src/path.cyr:258, 259, 305`), and the other `sd_alloc` sites listed below.
**Severity:** **Medium.** The 0.5.5 seam lets a consumer install any allocator; an arena that can
refuse is the natural one to install. Today a refusal inside path construction is a null-pointer
store, so the only safe hook is one that never returns 0 — which the global allocator itself cannot
promise (it returns 0 when its mmap fails).

## What happens

`sd_path_new` checks its 48 B record and nothing else:

```cyr
var p = sd_alloc(SD_PATH_SIZE);
if (p == 0) { return 0; }
var verbs = sd_alloc(SD_PATH_CAP * 8);    # unchecked
var points = sd_alloc(SD_PATH_CAP * 8);   # unchecked
```

A refused `verbs` block is stored as 0, and the first `sd_path_moveto` writes through it in
`sd_path_push_verb`. `sd_point_new` stores x/y through its allocation unchecked, on every point of
every path. (`sd_path_grow` does check both of its blocks.)

MEASURED (sadish 0.6.0): a hook that grants exactly one block and then returns 0 —
`sd_path_new()` returns a non-zero path, and `sd_path_moveto(path, 0, 0)` dies with **SIGSEGV, rc 139**.

Through rekha 0.3.11: under a hook that grants its first K blocks, a refusal landing on a rekha site
now returns 0 cleanly (rekha 0.3.10 faulted), but a refusal landing on sadish's verb buffer or an
`SdPoint` still faults the draw (rc 139).

## `sd_alloc` sites whose result is used without a 0 check

Found by a next-line scan of `src/*.cyr` (each worth confirming by eye):

| site | allocation | on the per-frame draw path |
|---|---|---|
| `path.cyr:48`, `:49` | `sd_path_new` verbs / points | yes |
| `geom.cyr:67` | `sd_point_new` | yes — every point |
| `path.cyr:258`, `:259`, `:305` | `sd_path_flatten` out / ctx / polyline | yes — every fill/stroke |
| `geom.cyr:96` | matrix | when transforming |
| `raster.cyr:736`, `:749` | clip node / shape mask | when clipping |
| `error.cyr:32` | error record | no |
| `present.cyr:69, 73, 114, 115, 129, 135` | PPM / presenter | no |

⚠ **Every `file:line` above and in the Affects line is AS FILED, against 0.6.0** — all 16 point at
unrelated lines at 0.7.1. Re-derived 2026-09-15: `sd_path_new` verbs/points `src/path.cyr:75`, `:77`
(via `_sd_path_alloc`); `sd_point_new` `src/geom.cyr:73`; `sd_path_flatten` out / ctx / polyline
`src/path.cyr:724`, `:730`, `:793`; matrix `src/geom.cyr:106` **and `:221`** — the tree has TWO matrix
allocations (`sd_matrix_new` and `sd_matrix_invert`), this table names one, and 0.7.1 checked both;
clip node / shape mask `src/raster.cyr:1018`, `:1057`; error record `src/error.cyr:41`; PPM /
presenter `src/present.cyr:90`, `:93`, `:146`, `:156`, `:176`, `:181`. Cite the FUNCTION, not the
line, in the next filing.

## Suggested fix

Check each site and propagate `SADISH_ERR_OOM` (or 0 for constructors), matching what `sd_path_grow`,
`sd_canvas_new` and the gradient constructors already do; `sd_path_push_point` / `sd_path_moveto`
already return a status a caller can test once `sd_point_new` can report failure. Then the seam's
contract can read "a hook may refuse" instead of "a hook must fall back", and rekha's own 0-checks
become end-to-end.

## Status — `raster.cyr` / `present.cyr` / `error.cyr` sites CLOSED in 0.7.1

⚠ **This section covers only half the filing.** The `path.cyr` / `geom.cyr` rows above
(`sd_path_new`, `sd_point_new`, `sd_path_flatten`, the matrix) are a separate 0.7.1 item, gated by
`programs/flatten_bound_test.cyr`. ⛔ It **landed in the same release** — this section was written
before it did, and used to end "the top-line **Status** stays open until that one lands too", which
contradicted the header two screens above for the whole of 0.7.1. Both halves are in; what keeps the
top-line Status open is a different site entirely (**Still open**, at the foot).

Closed here, each with a numbered check in the new suite `programs/oom_test.cyr` (157 checks):

| site | 0.7.0 | 0.7.1 |
|---|---|---|
| `raster.cyr` clip node (`sd_clip_push_mask`) | null-pointer store | `SADISH_ERR_OOM`, and the node is now taken **before** the in-place intersection, so a refusal leaves the caller's `shape` byte-for-byte untouched |
| `raster.cyr` shape mask (`sd_canvas_clip_push_rect`) | null-pointer store | `SADISH_ERR_OOM`, nothing written |
| `raster.cyr` temp canvas (`sd_canvas_clip_push_path`) | `sd_canvas_new` 0 → null load | `SADISH_ERR_OOM`; an OOM out of the mask fill is propagated (an EMPTY_PATH is not — it still clips to nothing, as it always did) |
| `raster.cyr` `_sd_fill_accrow_for` (found on re-scan) | stored 0 **and raised the capacity**, poisoning the scratch for the life of the process | keeps the old block and capacity; `sd_fill_impl` returns `SADISH_ERR_OOM` with the coverage buffer untouched |
| `raster.cyr` `_sd_scratch_init` (found on re-scan) | published a partial set with `_sd_fill_ectx` non-zero, so it never retried | all-or-nothing; `sd_fill_impl` returns `SADISH_ERR_OOM` and the next call re-tries |
| `present.cyr` PPM header + RGB | null-pointer store | `SADISH_ERR_OOM`, and **both blocks are taken before the `O_TRUNC` open**, so a refusal leaves the existing file exactly as it was |
| `present.cyr` presenter record + blit scratch + `vinfo` / `finfo` | null-pointer store / a presenter with a 0 scratch | 0, with the framebuffer **fd closed** on every one of the four paths |
| `error.cyr` `sadish_err_new` | null-pointer store | 0 — and a 0 record now READS: `sadish_err_code(0)` is `SADISH_ERR_OOM`, `sadish_err_detail(0)` is 0, so a caller handling one failure is never handed a second |

⭐ One deliberate behaviour change rides with the fix, and it is here rather than buried in a
comment: **a 0-area canvas is now `SADISH_ERR_OOM` out of `sd_canvas_clip_push_rect`** (MEASURED on
both trees with a hand-built `w = 0` header: 0.7.0 returned `SADISH_OK` and pushed a clip node whose
mask pointer was address 0; 0.7.1 refuses and pushes nothing). `sd_alloc(0)` is 0 by lib/alloc.cyr's
contract, and after 0.7.1 a null pointer does not enter a data structure. ⚠ Only a hand-built header
can reach it — `sd_canvas_new` refuses `w <= 0` and `h <= 0` — and no consumer builds one (rekha 23
suites, dhancha 18, all green). `src/present.cyr` deliberately answers the SAME `sd_alloc(0)` the
other way for a 0-pixel PPM (`SADISH_OK`, a header-only file, as through 0.7.0): that call completes
and leaves a valid file, while a 0-area clip push could only hand back the booby-trapped node. Both
answers, and `sd_canvas_clip_push_path`'s `SADISH_ERR_EMPTY_PATH`-is-not-propagated asymmetry, are
now pinned by checks (`oom_test` group K) instead of only stated in prose.

MEASURED, 0.7.1: a hook that refuses the K-th allocation, swept across every site above, returns the
documented code with the canvas's clip intact and poppable and the process running to completion —
`oom_test` prints PASS, rc 0.

⚠ The 0.7.0 comparison is stated the way it was actually obtained, because the suite **cannot be
built against 0.7.0** — it calls `_sd_present_probe` and `_sd_presenter_build`, which 0.7.1
introduces. What was measured instead: each 0.7.0 behaviour was reinstated in the 0.7.1 tree as a
single mutation and run against this suite. **Thirteen of those mutations, covering all twelve
unchecked sites above, are `SIGSEGV`, rc 139.** Two FURTHER 0.7.0 behaviours are not faults and are
killed by an assertion instead: reinstating `_sd_fill_accrow_for`'s capacity-on-refusal makes check
111 read 0 coverage where 16,320 is due, and reinstating the PPM's open-before-allocate order makes
checks 57-58 read a PPM truncated from 35 bytes to 0. Separately,
one site was measured end-to-end on a materialised 0.7.0 tree with a probe that builds against both
(`sd_canvas_clip_push_path` on a degenerate canvas header: 0.7.0 rc 139, 0.7.1 rc 1).

⛔ Rendering did not move: the 25 pre-0.7.1 suites pass, agnos's
`refagree` prints **BYTE-IDENTICAL on all 200 paths**, and rekha (23 suites) and dhancha (18) pass
against this `dist/`. ⚠ This used to add "with **assertions unedited**", which is over-broad for the
release it shipped in: `git show f722e8c -- programs/alloc_test.cyr` edits checks **#128** and **#133**
from `check(arena_used(g_arena) - pa, 1248)` to `208`, and rewrites `grow_edges_test`'s MEASURED heap
figures. Those are ALLOCATION-BYTE assertions moved by the sibling flatten filing, not coverage — so
"rendering did not move" survives; "assertions unedited" does not. (The 25/27 arithmetic is right:
both new suites arrived in `f722e8c`.)

⚠ Not closed by this half: `sd_present_open`'s own `open("/dev/fb0")` is still never exercised by a
test (that would write to the live display) — its four allocation guards are reached through
`_sd_present_probe`, which 0.7.1 split out precisely so a regular file's fd can drive them.
Re-verified 2026-09-15: no program calls `sd_present_open`; only `_sd_present_probe` /
`_sd_presenter_build` are driven (`oom_test` checks #69-#90).

---

## Still open (re-audited 2026-09-15 against 0.7.1, HEAD `f722e8c`)

⛔ **The same fault class, one level up, and sadish's own code is the caller.** `src/stroke.cyr:133`
`var rp = sd_path_new();` (`sd_stroke_seg`) and `src/stroke.cyr:145` `var dp = sd_path_new();`
(`sd_stroke_disc`) never test the 0 that 0.7.1 taught `sd_path_new` to return, and the next line is
`sd_path_moveto(rp, …)`. MEASURED on HEAD: a straight 2-point path, warm scratch, every `sd_alloc`
refused — `sd_canvas_stroke_path` dies **rc 139**. Adding `if (rp == 0) { return 0; }` and
`if (dp == 0) { return 0; }` in a copy makes the same call return rc 0; patching only `sd_stroke_seg`
still faults, so **both** sites are live. It also reaches `sd_canvas_stroke_path_ex` with
`SD_CAP_ROUND` + `SD_JOIN_ROUND`, which delegates at `src/stroke.cyr:1127-1129`.

⚠ Sweep of every public entry under a total-refusal hook on HEAD: fill, styled (butt/miter), dash,
`sd_canvas_clip_push_rect`, `sd_canvas_clip_push_path`, `sd_path_flatten`, `sd_path_new`,
`sd_path_moveto` all survive; **`sd_canvas_stroke_path`** and
**`sd_canvas_stroke_path_ex(ROUND, ROUND)`** SIGSEGV. No suite drives a refusing hook through any
stroke entry point — grep of `oom_test.cyr` and `flatten_bound_test.cyr` for `stroke` finds only the
scratch-pointer assertions #135-#138 — which is why this survived the release.

⚠ `src/stroke.cyr:545` `var accrow = _sd_fill_accrow_for(w);` — the styled span walk still stores
through a possible 0. `src/raster.cyr:400-401` admits it in the tree ("⚠ THE CALLER IN stroke.cyr …
faults on a refusal exactly as it did through 0.7.0, and that file is out of this change's scope").
The `_sd_fill_accrow_for` row of the table above names only `sd_fill_impl` as its consumer.

**The Suggested fix's second clause is not done.** `src/alloc.cyr:28-30` still carries the pre-fix
contract: *"A hook returning 0 is handled precisely as `alloc()` returning 0 already is: the callers
that check (`sd_surface_new`, `sd_canvas_new`, `sd_path_new`, `sd_path_grow`) return 0 /
`SADISH_ERR_OOM`; the callers that do not check never did. No new failure path."* That list is now
every caller in path/geom/raster/present/error, and the closing sentence is false. Its last clause —
"and rekha's own 0-checks become end-to-end" — cannot be true while a stroke faults, so the Severity
paragraph's *"the only safe hook is one that never returns 0"* still holds for any consumer that
strokes.

**What would close it:** two lines in `src/stroke.cyr` and one suite case (a stroke under `oom_test`'s
`oom_budget(0)`). Then the seam contract in `src/alloc.cyr:28-30` can finally be rewritten as this
filing asks.
