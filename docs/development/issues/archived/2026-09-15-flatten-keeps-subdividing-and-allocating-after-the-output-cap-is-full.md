# `sd_path_flatten` keeps subdividing — and allocating mid-points — after its 8,192-point output is full

> 📁 **ARCHIVED — the Status line below is current; everything after it is AS FILED.** In-body
> `file:line` citations, suite counts and any "this filing does not archive" line were true when
> written and are **not maintained**: re-pointing them would edit the measurement, and the
> measurement is the point. Where the body and the Status line disagree, the Status line wins.
> ⇒ For what is actually left, read [`../../roadmap.md`](../../roadmap.md).

**Status:** 🟢 **CLOSED in 0.8.0 — the BOUND shipped in 0.7.1, the FILL got the budget's scope and the
STROKES their return code in 0.7.2, and the POLYLINE its own verdict in 0.8.0. All three Still-open
items are closed.** ⛔ Item 2 was closed by 0.7.2 and this filing did not say so: the finding was
written against 0.7.1 and left standing verbatim rather than re-pointed — correct for a record, wrong
for a Status line. MEASURED 2026-09-16 on 0.8.0 (a 3 px-wide stroke of a cubic on 64x64 under a hook
granting K allocations): `sd_canvas_stroke_path` and `sd_canvas_stroke_path_ex` both return
`SADISH_ERR_OOM` at K = 0, 1, 2 and 3, where 0.7.1 returned `SADISH_OK` over a partial picture and the
round stroker faulted. Gated by `programs/stroke_oom_test.cyr` (153 checks). The mid-point waste, the cross-curve stop and the per-operation budget are
all in, and every figure in the closing section at the foot of this file was RE-MEASURED 2026-09-15
against 0.7.1 (HEAD `f722e8c`) and reproduced. ⭐ **0.7.2 closed Still-open item 3**, the one the
Severity line is about: `sd_fill_impl` now opens ONE flatten operation for the whole fill, so an
unwrapped `sd_canvas_fill_path` of an untrusted outline is bounded by the budget and
`sd_flatten_degraded()` answers for it (MEASURED: the hostile 4,096-quad fill 16,711,680 B →
**1,044,480 B**, 1,044,480 → **65,280** allocations, and the same figure at 1,024 quads — the work no
longer grows with the curve count). ⭐ **0.8.0 closed Still-open item 1**, the first half of the
second suggested fix: `SdPolyline` is 24 B and carries its own `truncated` / `degraded` verdict,
frozen when `sd_path_flatten` finishes, so a consumer holding a result can tell a complete flattening
from a PREFIX without polling a process-wide flag that has moved on. ⚠ The prefix is still RETURNED
rather than refused — the bullet's own "refused **or** clearly degraded" — and section 5 at the foot
says why, on the record. ⛔ **This filing is still NOT archivable**: item 2 is open, and this file's
own ⛔ note under it records that 0.7.2 already changed the tree it describes, so what is left of that
ask has to be re-derived against the shipped code rather than read off these words.
⛔ This line read "🟢 CLOSED — FIXED in sadish 0.7.1" until the 0.7.1 re-audit; a reader of the
Status line alone came away with the wrong picture.
Was 🟡 OPEN — MEASURED on sadish **0.6.0** (and identically on 0.5.5 and 0.7.0).
**Filed:** 2026-09-15, by **rekha** (0.3.11 audit — a fresh security review measured it through rekha's
draw path; re-measured here against sadish alone).
**Affects:** `src/path.cyr` — `sd_flat_emit`, `sd_flatten_quad`, `sd_flatten_cubic`, `sd_path_flatten`;
every fill and stroke of a curved path.
**Severity:** **High for any consumer that fills untrusted outlines** (rekha → dhancha → crab draw glyph
paths from font files every frame): the work of one fill is bounded by the path's CURVE COUNT, not by
`SD_FLATTEN_CAP`, while the OUTPUT is silently truncated at the cap.

## What happens

`sd_flat_emit` drops a point once `count >= cap` and returns 0 — and nothing upstream looks. So
`sd_flatten_quad` / `sd_flatten_cubic` go on subdividing to `SD_FLATTEN_MAX_DEPTH` (8) for every
remaining curve, and every subdivision node calls `sd_pt_mid` three (quad) or six (cubic) times,
each an `sd_point_new` — a 16 B `sd_alloc` — for points that are then thrown away.

Two consequences:

1. **Cost past the cap is unbounded by the cap.** Each maximally non-flat quad costs ~765
   allocations whether or not a single point of it can still be emitted.
2. **Truncation is silent.** A path needing more than 8,192 flattened points loses its tail; the
   polyline reports `count == cap` and the fill proceeds on an open contour.

## MEASURED (sadish 0.6.0, `sd_path_flatten(path, 1)` under a counting `sd_alloc` hook)

Path: `moveto(0,0)` then N × `quadto(1e9, 1e9, 0, 0)` (each needs the full depth-8 subdivision).

| quads | polyline points | bytes allocated | allocations |
|---:|---:|---:|---:|
| 32 | 8,192 (= cap) | 457,256 | 24,483 |
| 64 | 8,192 | 848,936 | 48,963 |
| 1,024 | 8,192 | 12,599,336 | 783,363 |
| 4,096 | 8,192 | **50,200,616** | **3,133,443** |

32 quads already fill the output; the other 4,064 quads of the last row add ~49.7 MB of mid-points
that are never used.

Through a real consumer (rekha 0.3.11, which now caps one glyph at 16,384 decoded points): a hostile
but cap-legal glyph — 4,096 off-curve points, alternating ±32767 x deltas, wrapped in three 2×2 (−2.0)
composite levels — costs **12,386,304 B, 774,144 allocations and ~47 ms for ONE
`sd_canvas_fill_path` on a 64×64 canvas** (sadish 0.6.0; 0.5.5 identical). The same glyph's path
itself is 930,104 B; the fill costs 13× the path. Per glyph, per label, per frame.

Repro (sadish-only):

```cyr
var path = sd_path_new();
sd_path_moveto(path, 0, 0);
var i = 0;
while (i < 4096) { sd_path_quadto(path, 1000000000, 1000000000, 0, 0); i = i + 1; }
# count sd_alloc calls with a hook around:
var pl = sd_path_flatten(path, 1);   # load64(pl + SD_POLYLINE_COUNT_OFFSET) == 8192
```

## Suggested fix

- Make "full" observable: `sd_flat_emit` returns a full flag, and `sd_flatten_quad` /
  `sd_flatten_cubic` check `count >= cap` **before** subdividing, returning without calling
  `sd_pt_mid`. That alone bounds the post-cap cost to one comparison per remaining curve.
- Report truncation: set a flag on the polyline (or return an error the fill/stroke can see) so a
  truncated contour is refused or clearly degraded, rather than filled open.
- Relation to README's candidate *"a growable edge list (lift `SD_FLATTEN_CAP`)"*: lifting the cap
  fixes truncation but makes this cost grow with the input in the other direction — whatever replaces
  the cap should still bound total flatten work (points emitted and mid-points allocated) per call.

rekha will add a regression case for this to its hostile corpus once sadish ships the bound, asserting
the fill's allocation count for the glyph above.

---

## CLOSED — sadish 0.7.1 (`src/path.cyr`, `programs/flatten_bound_test.cyr`)

Three separate changes, all in `src/path.cyr`. Every figure below is MEASURED on the 0.7.1 tree with
the counting `sd_alloc` hook this filing used, on the same repro.

**1. The recursion carries COORDINATES, not `SdPoint`s.** `sd_pt_mid` is no longer on the subdivision
path at all: `_sd_flat_quad_r` / `_sd_flat_cubic_r` compute their de Casteljau mids as plain i64
locals and hand them down as numbers, and only `_sd_flat_put` allocates — for a point that is
actually going into the output. Allocation is now exactly *(points emitted − 1)* per curve.
A maximally non-flat quad: **765 allocations → 255**. A cubic: **6 per node → 1**.
⇒ The suggested per-depth scratch was not needed, and is not there: there is no scratch to warm,
size or grow, and the one-time cost is 0 B (a scratch would have cost 720 B of global heap inside
`programs/alloc_test.cyr`'s group-B headline, which asserts that twenty frames of curve drawing
under an arena hook cost the global heap EXACTLY 0).

**2. "Full" is observable.** `sd_flat_emit` records the emit ctx that dropped a point in
`_sd_flat_full`, and both recursions test it before doing anything else. ⚠ It is the **ctx**, not a
flag: a first draft used a plain flag cleared only at nesting depth 0, and since `sd_path_flatten`
opens a NESTED operation whenever a consumer has one open — the usage this fix's own docs prescribe
— a starved flatten's flag survived into the next, healthy flatten of the same operation and
truncated it to its straight-verb anchors while `_sd_flat_trunc = 0` wiped the witness (MEASURED:
1 point of 513, with `truncated()` and `degraded()` both reading 0). Fullness is a property of the
OUTPUT ARRAY, and `sd_path_flatten` clears the verdict outright when it establishes its own ctx,
because an arena the consumer reset can hand the next call the same address.
⛔ And a ctx POINTER is not enough on its own either, because raster.cyr and stroke.cyr each own ONE
process-lifetime per-curve buffer and REWIND it before every curve verb: the ctx that was full a
moment ago is an empty output now, and inside a consumer's operation nothing cleared the verdict
against it. MEASURED before the second repair, one starved fill and then HEALTHY fills inside one
`sd_flatten_op_begin`/`_end`, 64×64, a 20 px circle of 4 cubics (true ink 312,280 coverage units):
every later fill in that operation returned `SADISH_OK` and painted a **completely blank canvas**,
with the allocator fully restored, for the whole life of the operation. ⇒ `_sd_flat_enter`
re-derives the verdict from the ctx's own `(count, cap)` at every entry to `sd_flatten_quad` /
`sd_flatten_cubic`: it survives only while the output REALLY has no room (which is what the
cross-curve stop below rests on), a rewound output starts clean, and `sd_path_flatten`'s own
accumulating ctx is exempt. A verdict set by a refused `SdPoint` rather than a full array costs one
descent and one refused allocation per curve to re-discover — bounded, and it cannot outlive the
starvation that caused it.
Post-full cost is one comparison per remaining curve:
MEASURED, 64 maximally non-flat quads into an arena that cannot double the output — the flatten makes
**8,164 allocation requests, exactly one of them refused**, and the 32 quads after the output filled
ask for nothing at all (0.7.0: 32 × 765 = 24,480 more requests for points nothing could hold).
The prefix is byte-identical to 0.7.0's.

**3. A per-operation budget in POINTS.** `SD_FLATTEN_BUDGET_DEFAULT = 65,536`, with
`sd_flatten_budget_set(n)` (returns the previous) / `sd_flatten_budget_get()`, 0 = unbounded — the
shape of raster.cyr's growth ceiling, in the unit this call allocates in. Over budget each remaining
curve emits its ENDPOINT — its chord — with no subdivision and no temporaries, and
`sd_flatten_degraded()` reports it to the consumer. The default's headroom, MEASURED at the fill's
own tolerance: a 256 px circle of cubics 65 points; the real LiberationSans printable-ASCII set
through `rekha_outline_to_sdpath` 2,532 points at 32 px (max 106 a glyph), 14,787 at 2,048 px (max
785); `grow_edges_test.cyr`'s `cubic_ring(3000)` — the largest curve path in these suites — 8,601.
⇒ 7.6× the largest suite path, 83× the largest 2,048 px glyph.

### The table, re-measured (`sd_path_flatten` of N maximally non-flat quads)

| quads | points 0.7.0 | 0.7.1 | bytes 0.7.0 | bytes 0.7.1 | allocs 0.7.0 | allocs 0.7.1 |
|---:|---:|---:|---:|---:|---:|---:|
| 32 | 8,193 | 8,193 | 588,328 | **327,208** | 24,484 | **8,164** |
| 64 | 16,385 | 16,385 | 1,242,152 | **719,912** | 48,965 | **16,325** |
| 1,024 | 262,145 | 66,305 | 20,856,872 | **3,076,136** | 783,369 | **65,287** |
| 4,096 | 1,048,577 | 69,377 | 83,623,976 | **3,076,136** | 3,133,451 | **65,287** |

⚠ (0.7.0's rows are this tree's own re-measurement — they are NOT the filing's rows. This line used to
say they "reproduce the filing exactly at 4,096 quads"; the filing's MEASURED table is **0.6.0**:
8,192 points, 50,200,616 B, 3,133,443 allocations. Re-measured 2026-09-15, only the allocation count
is close, and it differs by exactly 8 — the eight output doublings 0.7.0 added. The bytes differ by
1.67×, because 0.7.0 grew the output where 0.6.0 truncated it at the cap. The 0.7.0 column itself
reproduces exactly: 1,048,577 / 83,623,976 / 3,133,451.)
At 4,096 quads: **27.2× the bytes, 48.0× the calls**, and the work no longer grows with the curve
count at all — the 1,024 and 4,096 rows are the same figure, which is the budget.

### One `sd_canvas_fill_path` of the 4,096-quad path on 64×64

| | seam bytes | seam allocs |
|---|---:|---:|
| 0.7.0 | 50,135,040 | 3,133,440 |
| 0.7.1 | **16,711,680** | **1,044,480** |
| 0.7.1 inside `sd_flatten_op_begin/end` | **1,044,480** | **65,280** |
| **0.7.2, unwrapped** | **1,044,480** | **65,280** |

⚠ raster.cyr flattens ONE CURVE VERB at a time into a fixed per-curve buffer. Through 0.7.1 each of
those calls was its own flatten operation, so the budget bounded one curve there (≤ 256 points) and
not the fill; the 3× is the mid-point fix alone. A consumer that wanted a whole untrusted fill bounded
had to scope it with the public `sd_flatten_op_begin()` / `sd_flatten_op_end()` — the third row, 48×
under 0.7.0 — and read `sd_flatten_degraded()` afterwards.
⭐ **0.7.2 made that the default** (fourth row, MEASURED on the same repro): `sd_fill_impl` opens the
operation itself, so wrapping is now a no-op in cost rather than the difference between bounded and
unbounded. It is not the one-line change this paragraph used to predict — the open is LAZY, at the
first curve the fill really flattens, because `sd_flatten_degraded()`'s contract is that a fill of a
path with no curve verbs opens no operation and leaves the standing verdict alone (an unconditional
`sd_flatten_op_begin()` at the top of `sd_fill_impl` would clear a consumer's verdict on every
rectangle it draws — MEASURED as a mutation: it passes every other check in the suite and fails
`flatten_bound_test` #334 and #342). See Still open, item 3.

### 4. THE FILL IS ONE OPERATION (0.7.2) — Still-open item 3

`sd_fill_impl` opens a flatten operation at the first curve verb it really flattens and closes it when
the verb walk ends. Ten lines of code in three places — a flag, an open in each of the two curve
branches, one close — not the one line this file predicted, and the shape of them is the contract:
the open is inside the `qe != 0` / `ke != 0` guards (a curve before any moveto is skipped, so it opens
nothing), it happens at most once a fill (a flag, not per verb), and the close is paired with that
flag (so it can never end a level the CONSUMER opened). Nothing between the open and the close
returns, so the nesting depth cannot leak.

**Proof the gate bites** — MEASURED on the 0.7.2 tree, each mutation applied ALONE and reverted, with
`flatten_bound_test`, `clip_pitch_test`, `integration_test`, `alloc_test`, `grow_edges_test`,
`oom_test` and `area_test` re-run under each:

| mutation of the 0.7.2 change | what fails |
|---|---|
| the QUAD branch's open deleted | `flatten_bound_test` 102, 103, 104, 307, 308 — no other suite |
| the CUBIC branch's open deleted | 309, 310, 354, 355 — no other suite |
| the close deleted | 78 (got 33, want 98), then the suite dies (rc 139): the depth never returns to 0, so every later flatten is nested, the point count never resets, and group G's tail walk reads 98 points out of a polyline that now holds 33 |
| the open made unconditional (this file's "one-line change") | 334, 342 — the standing-verdict contract, and nothing else |
| the close made unconditional (`fop != 0` guard dropped) | 338 — it ends a level the consumer opened |
| `fop = 1` never set (an open per curve, one close) | 78, then rc 139, as above |
| the close moved below the closing edge-add | **EQUIVALENT** — `sd_edge_add` flattens nothing, so the operation covers the same flattens; all 7 suites stay green, as expected |

⛔⛔ **AND WHAT RE-MEASURING 102/103/104 TOOK AWAY, which a review caught and this filing records so
it cannot happen again.** `sd_flatten_quad` and `sd_flatten_cubic` open a flatten operation of their
OWN around each curve (`src/path.cyr`). Once `sd_fill_impl` opens one first, those two begins nest
inside it for a fill — so the pair is now the depth-0 operation for exactly ONE caller left:
`stroke.cyr`, which walks a path one curve verb at a time and never opens a scope (`_sd_round_walk`
`src/stroke.cyr:280`/`:294` and `_sd_styled_walk` `:1257`/`:1273`, at 0.7.2 — the round walk was
still inline in `sd_canvas_stroke_path` at 0.7.1, `:247`/`:261`; the stroker's own fills go through
`sd_canvas_fill_union` with lineto-only polygons, which open nothing). Nothing else resets
`_sd_flat_used` for an unwrapped stroke. Checks 102/103/104 were that pair's only gate, and
re-measuring them for the fill scope left it ungated: MEASURED on the 0.7.2 tree, deleting the
begin/end from `sd_flatten_quad` ALONE left **all 27 suites green**, and so did deleting both pairs.
The behaviour it guards is real — MEASURED at a budget of 64 points, ten identical unwrapped strokes
of one 4-cubic circle on a fresh 64×64 canvas: **126,930** coverage units every time with the pair,
and **126,930 four times then 114,300 with `degraded()` = 1 for ever after** without it, because
`_sd_flat_used` then accumulates for the life of the PROCESS. ⇒ `flatten_bound_test` **group P**
(checks 359-372) replaces the lost gate: 360/361 drive the plain stroker's cubic and quad branches,
362/363 the styled walk's, each ten strokes deep against a MEASURED ink figure and
`sd_flatten_degraded()` == 0; 364-369 are the anti-vacuity half (the same stroke repeated inside ONE
consumer operation shares one budget, so the fifth is cut to chords — the budget really is spendable
at 64); 370-372 restore the default and re-check the whole picture.

| mutation of code the re-measure un-pinned | what fails |
|---|---|
| `sd_flatten_quad`'s own begin/end deleted | `flatten_bound_test` 361, 363 — no other suite (before group P: nothing, in any of the 27) |
| `sd_flatten_cubic`'s own begin/end deleted | 360, 362, 372 — no other suite (before group P: nothing) |
| both deleted | 360, 361, 362, 363, 372 — no other suite (before group P: nothing) |

**Byte-identical**, as 0.7.1 was: all 27 suites pass with their assertions unedited except the three
group-H figures this deliberately moves (102: 16,711,680 → 1,044,480 B; 103: 1,044,480 → 65,280 calls;
104: `sd_flatten_degraded()` 0 → 1 — the verdict the unwrapped consumer used to be denied), agnos's
`refagree` prints *BYTE-IDENTICAL on all 200 paths* against the 0.7.2 `dist/`, and rekha's 23 suites
and dhancha's 18 pass against it with no `undefined function`.

### 5. THE POLYLINE CARRIES ITS OWN VERDICT (0.8.0) — Still-open item 1

`sd_path_flatten` now returns a 24 B `SdPolyline` — `points`, `count`, and a **verdict word at +16** —
and freezes two bits into it the moment its verb walk ends:

| accessor | 1 means |
|---|---|
| `sd_polyline_truncated(pl)` | points were LOST (a growth or an `SdPoint` was refused): the array is a PREFIX and the contour is open where it stops |
| `sd_polyline_degraded(pl)` | at least one curve of **this call** was cut to its chord by the budget |
| `sd_polyline_verdict(pl)` | the raw word: `SD_POLYLINE_TRUNCATED` (1) `\|` `SD_POLYLINE_DEGRADED` (2) |

**Why this was not already answered by the two globals**, which is the whole of the ask. Both are
sticky on purpose: `sd_flatten_truncated()` holds until the next `sd_path_flatten` and
`sd_flatten_degraded()` until the next operation opens at depth 0. That is right for "did anything go
wrong since?" and useless for "is THIS polyline whole?" — a consumer that flattens three paths and
then looks gets one answer for all three. MEASURED as `flatten_bound_test` checks **408-419**: four
results held at once while later flattens drive both globals to 0 and back to 1; every record still
reads what it read when it was made.

⚠ **`degraded` is the CALL's, not the operation's — one new module global, `_sd_flat_cut`.** Reading
`_sd_flat_degraded` at the end of the walk looks equivalent and is not: inside a consumer's
`sd_flatten_op_begin` scope a budget spent by an EARLIER flatten leaves it standing at 1, so a whole
result would stamp itself DEGRADED — a per-result verdict that is really a poll of a shared global,
i.e. the defect this closure exists to fix. The two subdivision recursions now set both flags;
`_sd_flat_cut` is cleared by `sd_path_flatten` at the top of every call and read once at the bottom,
and nothing else in the process touches it. `sd_flatten_degraded()` is byte-for-byte unchanged.

⛔ **The clear's POSITION is part of the contract, and a review caught that it was not gated.** The
recursions that set `_sd_flat_cut` are also reached by the public per-curve entries
`sd_flatten_quad` / `sd_flatten_cubic` — the road `sd_fill_impl` and the stroker take — so between
two `sd_path_flatten` calls a cut can be made by something that is not an `sd_path_flatten` at all.
Only a clear that runs BEFORE the walk keeps it out of the next polyline's verdict. MEASURED with
the clear moved to just after the verdict store: **all 29 suites stayed green** while a budget-cut
fill followed by an untouched `sd_path_flatten` handed back a record reading `SD_POLYLINE_DEGRADED`.
Groups Q1 and Q6 do not hold this — in both, the call doing the cutting is itself an
`sd_path_flatten`, which a displaced clear handles. **Group Q9 (checks 439-455) is what holds it**,
in both shapes a consumer actually produces: a degraded `sd_canvas_fill_path`, and a bare
`sd_flatten_quad` inside a consumer's own operation scope.

⭐ **THE PREFIX IS FLAGGED, NOT REFUSED — and that is a choice, taken on the bullet's own "refused
**or** clearly degraded".** Returning 0 for a truncated flatten was weighed and rejected for three
reasons: (a) it throws away a result callers use — the starved flatten of check 76 keeps 8,192 points
that are byte-identical to the whole flattening's first 8,192, which is exactly what a consumer
recovering from an out-of-memory frame wants to draw; (b) `0` already means two things here (empty
path, refused allocation) and a third meaning on the same sentinel would deepen the confusion this
release is repairing, not settle it; (c) "refuse" belongs at the DRAW, where a partial contour
actually gets filled open, and it is already there — `sd_canvas_fill_path` (0.7.1) and
`sd_canvas_stroke_path` / `_ex` / `_dash` (0.7.2) return `SADISH_ERR_OOM` rather than paint half an
outline and report success. ⛔ So a polyline is NOT self-validating: nothing in sadish refuses to
rasterize a truncated one. `if (sd_polyline_truncated(pl) != 0)` is the caller's, and it is one line.

**The record grew, and that is the whole cost.** `SD_POLYLINE_SIZE` 16 → 24 B: +8 B per
`sd_path_flatten` that returns a result, no extra allocation and no extra call. Affordable because
nothing embeds an `SdPolyline` — swept 2026-09-16 over `src/`, `programs/`, rekha, dhancha, agnos,
setu and the two repos that vendor `dist/sadish.cyr` (crab, puka): `sd_path_flatten` is the only
constructor, `src/path.cyr` the only writer, no `SdPolyline` is ever a field of another record or an
array element, and every reader outside that file goes through `sd_polyline_count` /
`sd_polyline_points`. No consumer repo names `SD_POLYLINE_*` or calls `sd_polyline_*` at all; crab's
vendored bundle carries its own 0.6-vintage copy of the whole thing and is unaffected until it
re-vendors. **Seven asserted figures move, all of them +8 B, and no allocation COUNT does:**

| check | 0.7.2 | 0.8.0 |
|---|---:|---:|
| `flatten_bound_test` 47 / 51 / 54 / 58 (D: one flatten's bytes, 32 / 64 / 1,024 / 4,096 quads) | 327,208 / 719,912 / 3,076,136 / 3,076,136 | **327,216 / 719,920 / 3,076,144 / 3,076,144** |
| `flatten_bound_test` 75 (F: the exactly-full arena) | 196,136 | **196,144** |
| `grow_edges_test` 45 / 79 (a flatten at or under 8,192 points) | 65,576 | **65,584** |
| `grow_edges_test` 76 (20,001 points: 3 arrays + ctx + header) | 458,792 | **458,800** |

⇒ On the largest flatten these suites measure, the whole 24 B header is **0.005 %** of the 458,800 B
the call costs, and the 8 B this release adds is **0.002 %**.

**Proof the gate bites** — MEASURED on this tree, each mutation applied ALONE and reverted, with a
full 29-suite sweep under each:

| mutation of the 0.8.0 change | what fails |
|---|---|
| the `store64(pl + SD_POLYLINE_VERDICT_OFFSET, …)` deleted | `flatten_bound_test` 386, 388, 390, 394, 396, 399, 405, 406, 407, 411, 412, 413, 421 — no other suite. ⚠ This row leaves the verdict word UNINITIALISED, so which checks fail is heap-content dependent in principle; re-measured three consecutive runs on this tree, identical each time |
| the QUAD recursion's `_sd_flat_cut = 1` deleted | 386, 388, 406, 407, 411, 413, 421 — no other suite |
| the CUBIC recursion's `_sd_flat_cut = 1` deleted | 390 — and nothing else, in any suite |
| the per-call clear (`_sd_flat_cut = 0`) deleted | 379, 381, 395, 396, 400, 410, 412, 414, 417, 418, 424, 444, 445, 452, 453 |
| the per-call clear MOVED from the top of `sd_path_flatten` to just after the verdict store — *the mutation a review found unguarded* | **444, 445, 452, 453** — and **nothing at all** before group Q9 existed: all 29 suites passed the mutant |
| the verdict stamped from `_sd_flat_degraded` — *the plausible wrong implementation* | **424 alone**, in any suite |
| the TRUNCATED bit dropped | 394, 396, 399, 405, 407, 412, 413 |
| `sd_polyline_degraded` returning the raw masked word (2, not 1) | 386, 390, 406, 421 |
| the DEGRADED bit assigned rather than added (so it eats TRUNCATED) | 405, 407, 413 |
| `SD_POLYLINE_SIZE` left at 16 while the verdict still stores at +16 | 47, 51, 54, 58, 75, 373, 410, 411, 412, 413, 417 **and** `grow_edges_test` 45, 76, 79 |
| the verdict taken AFTER the header allocation | **EQUIVALENT** — the only thing between the two points is this function's own refused `sd_alloc`, which returns 0 and hands back no record; all 29 suites stay green, as expected |

⛔ Still true of 0.8.0, and worth saying again: ten of those eleven mutations fail
`programs/flatten_bound_test.cyr` and (except the record size, which is arithmetic) **no other
suite**. It remains the single program this entire repair rests on — ⚠ and the displaced-clear row
is what that concentration costs when a gate is missing: one wrong line, 29 green suites, and the
only thing that noticed was a review reading the clear's position against its own header.

**Byte-identical**, as 0.7.1 and 0.7.2 were: all 29 suites pass with their assertions unedited except
the seven +8 B figures above, agnos's `refagree` prints *BYTE-IDENTICAL on all 200 paths* against a
`dist/` built from this tree, and rekha's 23 suites and dhancha's 18 pass against it with no
`undefined function`. `flatten_bound_test` 372 → **455** checks (group Q's 83).

⚠ **Re-derived lines for `src/path.cyr` at 0.8.0**, since this file's older paragraphs cite 0.7.2's.
⛔ The first cut of this table was **off by one on every entry** — a review re-derived it and found
0-based numbers in a 1-based citation, in the one filing that exists partly because references rot.
These are `grep -n` output, and the command is here so the next reader re-derives rather than trusts:

```sh
grep -n 'var SD_POLYLINE_\|^fn sd_polyline_\|^var _sd_flat_cut\|^fn sd_flat_emit\|^fn _sd_flat_enter\|^fn sd_path_flatten' src/path.cyr
```

The polyline constants are `:376-386` (were `:352-354`), the accessors `:388-417`, `_sd_flat_cut`
`:510`, `sd_flat_emit`'s unconditional `return 0;` `:644` (was `:549`), `_sd_flat_enter`'s
`(count, cap)` test `:689` (was `:603`), `sd_path_flatten` `:909`. ⭐ Cite the FUNCTIONS; these will
rot too — this table rotted once before it shipped.

### Geometry

⛔ Byte-identical. `programs/flatten_bound_test.cyr` carries 0.7.0's recursion verbatim — its own
`sd_pt_mid`, its own copies of both flatness predicates — and compares raw coordinate against raw
coordinate for depth-8 quads and cubics, circles of cubics from 16 to 4,096 px radius, negative and
sub-pixel coordinates, tolerances 1 / SD_ONE / 0 / negative, `cubic_ring(3000)` and all 95
ASCII-shaped glyph paths; and the emitted `SdPoint` POINTERS for a curve's endpoints are still the
path's own. agnos's `refagree` oracle reports BYTE-IDENTICAL on all 200 paths.

Truncation remains reportable (`sd_flatten_truncated()`, `_sd_flat_trunc`); degradation is the new,
separate verdict (`sd_flatten_degraded()`).
⛔ This sentence used to end "unchanged since 0.6.1", and both halves were wrong. **There is no 0.6.1**
in this repo — no tag (0.6.0 then 0.7.0) and no CHANGELOG section; `_sd_flat_trunc` and the growable
output shipped in **0.7.0** (`536a3b4`). ⭐ The same phantom label sat in `src/path.cyr` twice and in 22
other places across `src/` and `programs/`; all 24 were corrected to 0.7.0 in `cc6fd75`, after this
paragraph was written. And it is not unchanged: the PUBLIC accessor `sd_flatten_truncated()` does not exist anywhere
in 0.7.0's src (`git show 536a3b4:src/path.cyr` — 0 hits), it is **new in 0.7.1**; `_sd_flat_trunc`
gained a set site in `_sd_flat_put`, had its clear moved before the empty-path return in
`sd_path_flatten`, and is now saved and restored around a fill by raster.cyr.

### ⚠ What the two verdicts do and do not answer for

- `sd_flatten_truncated()` belongs to the last **call** and is cleared by `sd_path_flatten` alone.
  It is also SET by the per-curve flattens inside a fill or a stroke. ⛔ **This bullet used to say it
  is the ONLY witness a starved fill leaves, "because `sd_canvas_fill_path` cannot see a refused
  mid-point and returns `SADISH_OK`". That is no longer true of the shipped code** — the fix landed in
  the SAME release, out of the sibling filing
  `2026-09-15-path-construction-stores-through-a-refused-allocation.md`: `sd_fill_impl` scopes the
  flatten's truncation verdict to the call and returns **`SADISH_ERR_OOM`** when that fill lost points
  (`src/raster.cyr` `sd_fill_impl`: `flat_was` at its verb walk, `flat_lost` / `flat_rc` after it —
  `:606` and `:676-679` at 0.7.1, `:643` and `:727-730` at 0.7.2, which is why this file's own README
  says to cite the FUNCTION), pinned by `programs/flatten_bound_test.cyr` check #271 (the
  assertion that changed with it, `SADISH_OK` → `SADISH_ERR_OOM`). The ink figures stand as MEASURED:
  a 20 px circle of 4 cubics on 64×64 under a hook granting 5 allocations paints 81,029 of 312,280
  coverage units with `truncated()` = 1 — it now says so in its return code too. (0.7.0 did not get
  that far: it dereferenced the refused `SdPoint` and died.) ⚠ The same stale sentence is in the
  shipped source comment — `sd_flatten_truncated`'s own `@public` header in `src/path.cyr`
  (`:408-412` at 0.7.1; that header is `:466-479` at 0.7.2) — and `cyrius distlib` copies it into
  `dist/sadish.cyr`, where a consumer reads it. ⛔ **That claim is STALE as of 0.7.2, checked
  2026-09-16 — the pointer above is updated, the sentence is left standing.** 0.7.2 rewrote the
  header, which now opens *"IT IS NO LONGER THE ONLY WITNESS, and the header said it was through
  0.7.1"*; `grep -rn "do not clear it"` over `src/*.cyr` and `dist/sadish.cyr` returns **0 hits**, so
  the bundle no longer carries it either. ⚠ Nothing was folded into `_sd_fill_trunc`, and there is
  still no public fill-truncation accessor; raster.cyr took the per-call scoping route instead.
- `sd_flatten_degraded()` belongs to the last **operation**, and is cleared when the next one opens
  at nesting depth 0. Every `sd_path_flatten` is an operation — an empty path included, which is why
  both verdicts are cleared BEFORE the empty-path return — and since 0.7.2 so is every FILL that
  flattens at least one curve (0.7.1: every single curve verb, separately). ⚠ A fill of a path with
  **no curve verbs** opens none at all, so it leaves whichever verdict was already standing. A
  consumer that wants an answer for one particular draw scopes it with `sd_flatten_op_begin()` /
  `sd_flatten_op_end()`. ⭐ That last sentence had **no check in the suite** through 0.7.1 —
  re-verified by hand 2026-09-15, so the behaviour held, but a maintainer could break it silently.
  **0.7.2 pinned it**: `flatten_bound_test` group O checks **331-334** (a `degraded()` == 1 verdict
  established by a budgeted flatten survives a lineto-only fill that really fills 261,120 coverage
  units), **340-342** (the same for a path whose only curve verbs sit before its first moveto, which
  are skipped and so flatten nothing), **335-339** (that fill neither opens nor closes a level of a
  consumer's own scope: begin, begin, fill, end → 1, end → 0) and **343-344** (the other half — a
  fill that DOES flatten a curve answers for itself and clears what it inherited).
- ⭐ **There is a THIRD verdict since 0.8.0, and it is the only one that is not a global.**
  `sd_polyline_truncated(pl)` / `sd_polyline_degraded(pl)` belong to one RESULT and never move once
  `sd_path_flatten` has returned it. Reach for them when you hold a polyline; reach for the two
  globals when you do not — after a fill or a stroke, or when `sd_path_flatten` returned 0 and you
  need to know whether that was an empty path or a refused allocation. See section 5 above.

---

## Still open — EMPTY: all three closed (audited 2026-09-15 against 0.7.1; re-checked 2026-09-16 against 0.8.0)

Three of the "Suggested fix" asks above were not met by the shipped code. Each is MEASURED, and each
was already stated somewhere in the body — the Status line is what did not say so. **All three are now
closed: item 3 by 0.7.2, item 2 by 0.7.2, item 1 by 0.8.0.** The findings are kept verbatim as written
against 0.7.1, because a filing records what was measured; the ⛔ notes say what the tree does now.

1. ~~**"a truncated contour is refused or clearly degraded, rather than filled open"** — there is no
   flag on the polyline~~ — **CLOSED in 0.8.0** (`src/path.cyr` `sd_path_flatten` and the
   `sd_polyline_*` accessors, gated by `programs/flatten_bound_test.cyr` group Q, checks 373-455).
   The finding stood as follows. `sd_fill_impl` (`src/raster.cyr:595` at 0.7.2) said so in the tree:
   *"The canvas still holds that partial picture — the caller decides whether to clear and retry."*
   Nothing was refused. There was no flag on the polyline either: `SdPolyline` was 16 B, points +
   count (`SD_POLYLINE_*_OFFSET`, `src/path.cyr:352-354` at 0.7.2). What had shipped was a sticky
   global verdict plus a return code.
   **What 0.8.0 does:** the record is 24 B with a verdict word at +16, and `sd_polyline_truncated(pl)`
   / `sd_polyline_degraded(pl)` / `sd_polyline_verdict(pl)` read it. See section 5 at the foot for the
   measurement, the mutation table, and the reasoning behind FLAGGING the prefix rather than refusing
   it — which is the half of the bullet this closure deliberately does not take.
   ⚠ **What it does not do**, so nobody has to re-derive it: `sd_canvas_fill_path` still fills a
   truncated contour open and returns `SADISH_ERR_OOM` about it (the 0.7.1 behaviour, unchanged);
   nothing in sadish refuses to rasterize a flagged polyline; and the two zeros `sd_path_flatten`
   returns — empty path, refused allocation — are still told apart by `sd_flatten_truncated()` alone,
   because neither has a record to carry a verdict.
2. ~~**The `/stroke` half of the same bullet**~~ — **CLOSED in 0.7.2** (`sd_canvas_stroke_path` / `_ex`
   / `_dash` scope the flatten verdict per call and return `SADISH_ERR_OOM`; gated by
   `programs/stroke_oom_test.cyr`). ⚠ Like the fill, a starved stroke still PAINTS its prefix and
   reports rather than refusing — the same deliberate choice item 1 explains.
   The finding stood as follows. `sd_fill_impl`'s note
   (`src/raster.cyr:597` at 0.7.1): *"Strokes are NOT covered by this return."*
   `sd_canvas_fill_union`'s result is discarded in `sd_stroke_seg` and `sd_stroke_disc`
   (`src/stroke.cyr:139` and `:156` at 0.7.1), so a stroke of a starved path still returns success.
   ⛔ **The tree says neither of those things any more, checked 2026-09-16 — this is 0.7.1's finding,
   and it is left standing rather than re-pointed.** 0.7.2 rewrote the note to *"Strokes were NOT
   covered by this return through 0.7.1 … Since 0.7.2 each stroke entry scopes the verdict the same
   way and returns SADISH_ERR_OOM itself"* (`sd_fill_impl`, `src/raster.cyr:599-602`), and
   `sd_stroke_seg` / `sd_stroke_disc` now RETURN `sd_canvas_fill_union`'s result
   (`src/stroke.cyr:178`, `:201`). The ask IS closed, re-verified 2026-09-16 on 0.8.0: both entry
   points answer `SADISH_ERR_OOM` at every K of a starved cubic stroke.
3. ~~**"whatever replaces the cap should still bound total flatten work per call"**~~ — **CLOSED in
   0.7.2** (`src/raster.cyr` `sd_fill_impl`, gated by `programs/flatten_bound_test.cyr` group O and
   by group H's re-measured 102/103/104).
   The finding stood as follows. Bounded for `sd_path_flatten`, NOT for a fill by default, which is
   the path the Severity line is about (rekha → dhancha → crab, glyph outlines from font files every
   frame): a fill opened ONE operation per curve verb, so the budget bounded one curve (≤ 256 points),
   not the fill. MEASURED 2026-09-15 on 0.7.1, an unwrapped `sd_canvas_fill_path` of the hostile path:
   261,120 allocations / 4,177,920 B at 1,024 quads and 1,044,480 / 16,711,680 B at 4,096 — exactly
   linear in the curve count at 255 allocations a curve. And the unwrapped consumer got no warning:
   `sd_flatten_degraded()` read **0** after both (asserted as intended at check #104).
   **RE-MEASURED on 0.7.2**, same repro, same hook, the fill now scoping its own operation:

   | one unwrapped `sd_canvas_fill_path`, 64x64 | 0.7.1 | 0.7.2 |
   |---|---:|---:|
   | 1,024 maximally non-flat quads | 4,177,920 B / 261,120 | **1,044,480 B / 65,280** |
   | 4,096 maximally non-flat quads | 16,711,680 B / 1,044,480 | **1,044,480 B / 65,280** |
   | 1,024 depth-8 cubics | 3,784,704 B / 236,544 | **1,044,096 B / 65,256** |
   | 4,096 depth-8 cubics | 15,138,816 B / 946,176 | **1,044,096 B / 65,256** |
   | `sd_flatten_degraded()` after it | 0 | **1** |

   ⇒ 16.0× the bytes at 4,096 quads, 14.5× for the cubics, and the 1,024 and 4,096 rows are now the
   same figure — which is what "bounded" means. Wrapping in `sd_flatten_op_begin`/`_end` still works
   and now costs exactly the same, so a consumer that already scopes its draws is unaffected.
   ⚠ **The trade, decided on measurement rather than taste.** A path whose ONE fill emits more than
   `SD_FLATTEN_BUDGET_DEFAULT` = 65,536 points now degrades its remaining curves to chords mid-fill.
   Points emitted by one fill, MEASURED on this tree at the fill's own tolerance (`SD_ONE >> 2`):
   a circle of 4 cubics from 8 px to 4,096 px radius **16 → 256**; a rounded rect from 200x100 r=16 to
   1920x1080 r=240 **16 → 64**; a 64-cubic blob at radius 1,024 px **1,536**; these suites' own curve
   paths (area, stroke_style, mixed) **14 → 26**; `grow_edges_test`'s `cubic_ring(3000)`, the largest
   curve path in the tree, **8,600**; one synthetic ASCII glyph from a ~13 px em to a ~2,048 px em
   **60 → 960**; all 95 of them as ONE path — a whole label in one fill — **2,396 → 26,394**.
   ⇒ The largest legitimate fill anywhere in this repo or its glyph corpus spends **40 %** of the
   budget, the largest single glyph 1.5 %, a UI shape under 3 %. The default degrades a ONE-PATH label
   at about 1,889 glyphs at a 32 px em, 649 at 208 px, 236 at 2,048 px — per PATH, not per draw.
   **The default is therefore unchanged at 65,536**, which also keeps `sd_path_flatten`'s own figures
   (the table above) and README's published constant true.
   ⚠ A starved fill is still `SADISH_ERR_OOM` and a degraded one is still `SADISH_OK` plus
   `sd_flatten_degraded()` — degrading is not losing (group O checks 353-358; group M6 unedited).

⚠ Bullet 1's letter, for anyone auditing site by site: `sd_flat_emit` does not "return a full flag" —
it still `return 0;` unconditionally (`src/path.cyr:549` at 0.7.2) and the verdict travels in the
module-global `_sd_flat_full`; the pre-subdivision test is `_sd_flat_full == ctx`, not
`count >= cap`. `(count, cap)` is consulted only by `_sd_flat_enter` (`src/path.cyr:603` at 0.7.2),
when a verdict is already standing. Consequence the body does not state: a FOREIGN ctx handed in
already full with no verdict standing costs one descent to the first leaf and one dropped point per
curve, not one comparison. Bounded, and gated as intended (checks #290 / #301: 3 calls, then 1).

⛔ One structural note from the re-audit: `programs/flatten_bound_test.cyr` (304 checks at 0.7.1, **372
at 0.7.2** — group O's 54 and group P's 14) is the **only** suite that failed for any of 20 one-site
reversions of this repair — including the `src/raster.cyr` one. The bound, both verdicts and the
geometry byte-identity all rest on that single program; nothing in the other 26 suites would notice if
it were deleted. ⚠ Still true of 0.7.2: of the seven one-site mutations of the fill scope, **six fail
this suite and no other** (the seventh is equivalent), and so do all three mutations of the stroker's
per-curve operation (group P) — a full 27-suite sweep was run under each.

**Downstream:** *"rekha will add a regression case for this to its hostile corpus once sadish ships
the bound"* — unverifiable from this repo, and nothing here tracks whether it happened.
