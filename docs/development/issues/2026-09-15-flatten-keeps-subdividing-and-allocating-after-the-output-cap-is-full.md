# `sd_path_flatten` keeps subdividing — and allocating mid-points — after its 8,192-point output is full

**Status:** 🟡 **OPEN — MEASURED** on sadish **0.6.0** (and identically on 0.5.5).
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
