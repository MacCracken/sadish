# `sd_path_flatten` keeps subdividing — and allocating mid-points — after its 8,192-point output is full

**Status:** 🟢 **CLOSED — FIXED in sadish 0.7.1** (see the closing section at the foot of this file).
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

(0.7.0's rows are this tree's own re-measurement and reproduce the filing exactly at 4,096 quads.)
At 4,096 quads: **27.2× the bytes, 48.0× the calls**, and the work no longer grows with the curve
count at all — the 1,024 and 4,096 rows are the same figure, which is the budget.

### One `sd_canvas_fill_path` of the 4,096-quad path on 64×64

| | seam bytes | seam allocs |
|---|---:|---:|
| 0.7.0 | 50,135,040 | 3,133,440 |
| 0.7.1 | **16,711,680** | **1,044,480** |
| 0.7.1 inside `sd_flatten_op_begin/end` | **1,044,480** | **65,280** |

⚠ raster.cyr flattens ONE CURVE VERB at a time into a fixed per-curve buffer, so each of those calls
is its own flatten operation and the budget bounds one curve there (≤ 256 points), not the fill. The
3× is the mid-point fix alone. A consumer that wants a whole untrusted fill bounded scopes it with
the public `sd_flatten_op_begin()` / `sd_flatten_op_end()` — the third row, 48× under 0.7.0 — and
reads `sd_flatten_degraded()` afterwards. Giving the fill that scope by default is a one-line change
inside `sd_fill_impl`, which this repair did not touch.

### Geometry

⛔ Byte-identical. `programs/flatten_bound_test.cyr` carries 0.7.0's recursion verbatim — its own
`sd_pt_mid`, its own copies of both flatness predicates — and compares raw coordinate against raw
coordinate for depth-8 quads and cubics, circles of cubics from 16 to 4,096 px radius, negative and
sub-pixel coordinates, tolerances 1 / SD_ONE / 0 / negative, `cubic_ring(3000)` and all 95
ASCII-shaped glyph paths; and the emitted `SdPoint` POINTERS for a curve's endpoints are still the
path's own. agnos's `refagree` oracle reports BYTE-IDENTICAL on all 200 paths.

Truncation remains reportable (`sd_flatten_truncated()`, `_sd_flat_trunc`, unchanged since 0.6.1);
degradation is the new, separate verdict (`sd_flatten_degraded()`).

### ⚠ What the two verdicts do and do not answer for

- `sd_flatten_truncated()` belongs to the last **call** and is cleared by `sd_path_flatten` alone.
  It is also SET by the per-curve flattens inside a fill or a stroke — and it is the **only** witness
  a starved fill leaves, because `sd_canvas_fill_path` cannot see a refused mid-point and returns
  `SADISH_OK`: MEASURED, a 20 px circle of 4 cubics on 64×64 under a hook granting 5 allocations —
  `SADISH_OK`, 81,029 of 312,280 coverage units, `truncated()` = 1. (0.7.0 did not get that far: it
  dereferenced the refused `SdPoint` and died.) Folding that into `_sd_fill_trunc` belongs with
  raster.cyr.
- `sd_flatten_degraded()` belongs to the last **operation**, and is cleared when the next one opens
  at nesting depth 0. Every `sd_path_flatten` is an operation — an empty path included, which is why
  both verdicts are cleared BEFORE the empty-path return — and so is every single curve verb a fill
  flattens. ⚠ A fill of a path with **no curve verbs** opens none at all, so it leaves whichever
  verdict was already standing. A consumer that wants an answer for one particular draw scopes it
  with `sd_flatten_op_begin()` / `sd_flatten_op_end()`.
