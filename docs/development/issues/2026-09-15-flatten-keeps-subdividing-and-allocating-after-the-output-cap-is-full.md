# `sd_path_flatten` keeps subdividing — and allocating mid-points — after its 8,192-point output is full

**Status:** 🟡 **OPEN — the BOUND shipped in 0.7.1; the second suggested fix did not.** The mid-point
waste, the cross-curve stop and the per-operation budget are all in, and every figure in the closing
section at the foot of this file was RE-MEASURED 2026-09-15 against 0.7.1 (HEAD `f722e8c`) and
reproduced. What is NOT in: a truncated contour is still **filled open**, there is no flag on the
polyline, STROKES get no error at all, and an unwrapped `sd_canvas_fill_path` of an untrusted outline
— the path this filing's own Severity line is about — is still not bounded by the budget. See **Still
open** at the foot. ⛔ This line read "🟢 CLOSED — FIXED in sadish 0.7.1" until that re-audit; a reader
of the Status line alone came away with the wrong picture.
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

Truncation remains reportable (`sd_flatten_truncated()`, `_sd_flat_trunc`); degradation is the new,
separate verdict (`sd_flatten_degraded()`).
⛔ This sentence used to end "unchanged since 0.6.1", and both halves were wrong. **There is no 0.6.1**
in this repo — no tag (0.6.0 then 0.7.0) and no CHANGELOG section; `_sd_flat_trunc` and the growable
output shipped in **0.7.0** (`536a3b4`), and the same phantom label sits in `src/path.cyr:305-307` and
`:698`. And it is not unchanged: the PUBLIC accessor `sd_flatten_truncated()` does not exist anywhere
in 0.7.0's src (`git show 536a3b4:src/path.cyr` — 0 hits), it is **new in 0.7.1**; `_sd_flat_trunc`
gained a set site in `_sd_flat_put` (`src/path.cyr:498`), had its clear moved before the empty-path
return (`:713`), and is now saved and restored around a fill by raster.cyr.

### ⚠ What the two verdicts do and do not answer for

- `sd_flatten_truncated()` belongs to the last **call** and is cleared by `sd_path_flatten` alone.
  It is also SET by the per-curve flattens inside a fill or a stroke. ⛔ **This bullet used to say it
  is the ONLY witness a starved fill leaves, "because `sd_canvas_fill_path` cannot see a refused
  mid-point and returns `SADISH_OK`". That is no longer true of the shipped code** — the fix landed in
  the SAME release, out of the sibling filing
  `2026-09-15-path-construction-stores-through-a-refused-allocation.md`: `sd_fill_impl` scopes the
  flatten's truncation verdict to the call and returns **`SADISH_ERR_OOM`** when that fill lost points
  (`src/raster.cyr:606-606`, `:676-679`), pinned by `programs/flatten_bound_test.cyr` check #271 (the
  assertion that changed with it, `SADISH_OK` → `SADISH_ERR_OOM`). The ink figures stand as MEASURED:
  a 20 px circle of 4 cubics on 64×64 under a hook granting 5 allocations paints 81,029 of 312,280
  coverage units with `truncated()` = 1 — it now says so in its return code too. (0.7.0 did not get
  that far: it dereferenced the refused `SdPoint` and died.) ⚠ The same stale sentence is in the
  shipped source comment at `src/path.cyr:408-412`, and `cyrius distlib` copies it into
  `dist/sadish.cyr`, where a consumer reads it. ⚠ Nothing was folded into `_sd_fill_trunc`, and there
  is still no public fill-truncation accessor; raster.cyr took the per-call scoping route instead.
- `sd_flatten_degraded()` belongs to the last **operation**, and is cleared when the next one opens
  at nesting depth 0. Every `sd_path_flatten` is an operation — an empty path included, which is why
  both verdicts are cleared BEFORE the empty-path return — and so is every single curve verb a fill
  flattens. ⚠ A fill of a path with **no curve verbs** opens none at all, so it leaves whichever
  verdict was already standing. A consumer that wants an answer for one particular draw scopes it
  with `sd_flatten_op_begin()` / `sd_flatten_op_end()`. ⚠ That last sentence has **no check in the
  suite** — re-verified by hand 2026-09-15 (`degraded()` == 1 survives a lineto-only fill; a curve
  fill clears it), so the behaviour holds, but a maintainer can break it silently.

---

## Still open (re-audited 2026-09-15 against 0.7.1, HEAD `f722e8c`)

Three of the "Suggested fix" asks above are not met by the shipped code. Each is MEASURED, and each is
already stated somewhere in the body — the Status line is what did not say so.

1. **"a truncated contour is refused or clearly degraded, rather than filled open"** — it is still
   filled open. `src/raster.cyr:593` says so in the tree: *"The canvas still holds that partial
   picture — the caller decides whether to clear and retry."* Nothing is refused. There is no flag on
   the polyline either: `SdPolyline` is still 16 B, points + count (`src/path.cyr:288-294`). What
   shipped is a sticky global verdict plus a return code.
2. **The `/stroke` half of the same bullet** — not done. `src/raster.cyr:597-597`: *"Strokes are NOT
   covered by this return."* `sd_canvas_fill_union`'s result is discarded at `src/stroke.cyr:139` and
   `:156`, so a stroke of a starved path still returns success.
3. **"whatever replaces the cap should still bound total flatten work per call"** — bounded for
   `sd_path_flatten`, NOT for a fill by default, which is the path the Severity line is about
   (rekha → dhancha → crab, glyph outlines from font files every frame). A fill opens ONE operation
   per curve verb, so the budget bounds one curve (≤ 256 points), not the fill. MEASURED 2026-09-15:
   an unwrapped `sd_canvas_fill_path` of the hostile path costs 261,120 allocations / 4,177,920 B at
   1,024 quads and 1,044,480 / 16,711,680 B at 4,096 — still exactly linear in the curve count at 255
   allocations per curve, i.e. this filing's consequence 1 survives for fills at 1/3 the old cost.
   Wrapped in `sd_flatten_op_begin`/`_end` it is 65,280 at both sizes. ⚠ And the unwrapped consumer
   gets no warning: `sd_flatten_degraded()` reads **0** after both unwrapped fills (asserted as
   intended at check #104). The consumer has to KNOW to scope it. The ⚠ block above names the fix —
   "a one-line change inside `sd_fill_impl`, which this repair did not touch".

⚠ Bullet 1's letter, for anyone auditing site by site: `sd_flat_emit` does not "return a full flag" —
it still `return 0;` unconditionally (`src/path.cyr:475-488`) and the verdict travels in the
module-global `_sd_flat_full`; the pre-subdivision test is `_sd_flat_full == ctx`, not
`count >= cap`. `(count, cap)` is consulted only by `_sd_flat_enter` (`src/path.cyr:531`), when a
verdict is already standing. Consequence the body does not state: a FOREIGN ctx handed in already full
with no verdict standing costs one descent to the first leaf and one dropped point per curve, not one
comparison. Bounded, and gated as intended (checks #290 / #301: 3 calls, then 1).

⛔ One structural note from the re-audit: `programs/flatten_bound_test.cyr` (304 checks) is the **only**
suite that failed for any of 20 one-site reversions of this repair — including the `src/raster.cyr`
one. The bound, both verdicts and the geometry byte-identity all rest on that single program; nothing
in the other 26 suites would notice if it were deleted.

**Downstream:** *"rekha will add a regression case for this to its hostile corpus once sadish ships
the bound"* — unverifiable from this repo, and nothing here tracks whether it happened.
