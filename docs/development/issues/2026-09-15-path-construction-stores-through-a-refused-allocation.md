# Path construction stores through a refused `sd_alloc` — a hook that returns 0 faults inside sadish

**Status:** 🟡 **OPEN — MEASURED** on sadish **0.6.0**.
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

## Suggested fix

Check each site and propagate `SADISH_ERR_OOM` (or 0 for constructors), matching what `sd_path_grow`,
`sd_canvas_new` and the gradient constructors already do; `sd_path_push_point` / `sd_path_moveto`
already return a status a caller can test once `sd_point_new` can report failure. Then the seam's
contract can read "a hook may refuse" instead of "a hook must fall back", and rekha's own 0-checks
become end-to-end.
