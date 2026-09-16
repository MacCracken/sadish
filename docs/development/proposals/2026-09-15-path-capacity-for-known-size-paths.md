# Capacity-sized paths: `sd_path_new_cap(n_verbs, n_points)` for callers that know the path's size

**Status:** 🟡 **OPEN — a capability request.**
**Filed:** 2026-09-15, by **rekha** (dhancha's 2026-09-13 arena filing already named this cost: *"4,144 B
of which is `sd_path_new`'s opening capacity — a sadish/rekha question"*).
**Placement:** `src/path.cyr`, beside `sd_path_new`.

## The consumer

rekha converts a decoded glyph outline to an `SdPath` per glyph, per label, per frame (dhancha's
`dh_draw_text_ink` → `rekha_char_to_sdpath`). When it does, it knows the path's exact size before the
first `moveto`: a TrueType contour of n points emits at most n + 1 verbs and 2n + 1 points, and rekha
already walks the on-curve flags that decide it. `sd_path_new` ignores all of that and opens every
path at `SD_PATH_CAP` = 256 verbs + 256 points (4,096 B of arrays + the 48 B record), then doubles and
copies — abandoning the old arrays on a no-free arena — for anything bigger.

## MEASURED (rekha 0.3.11 on sadish 0.5.5 — sadish 0.6.0's `sd_path_new` is the same code)

LiberationSans, every printable ASCII glyph (U+0020..U+007E, 95 paths), `rekha_outline_to_sdpath`:

| | bytes |
|---|---:|
| paths today (`sd_path_new` + pushes) | **433,648** |
| the same paths at exact capacity (48 + 8·verbs + 8·points + 16·points) | **78,656** |

1,768 verbs and 2,498 points in total; the largest glyph needs 61 verbs. ⇒ **5.5× less** arena per
ASCII glyph (4,565 B → 828 B). rekha's `bench_hotpath` puts a 54-character label at 231,928 B of arena
per draw today; paths are ~92 % of it.

The other end: a 4,096-point composite outline grows 256 → … → 8,192 and abandons 126,976 B of
intermediate arrays on the arena per path (rekha 0.3.11 audit).

## What rekha needs

- `sd_path_new_cap(n_verbs, n_points)` — allocate the verb and point arrays at exactly the given
  capacity (0 → a sensible minimum), same `SdPath` layout, same growth on overflow, so an
  underestimate stays correct and an exact estimate never grows. `sd_path_new()` stays as is (rekha's
  alloc_test pins today's 4,192 B small-path cost; rekha updates that test when it adopts the new call).
- Checked allocations (see the companion issue
  `issues/2026-09-15-path-construction-stores-through-a-refused-allocation.md`).

## Worth measuring alongside (not requested here)

`SdPath` stores POINTERS to 16 B `SdPoint`s, so every point costs 24 B and one `sd_alloc` call — 39,968
of the 78,656 exact-capacity bytes above are `SdPoint` objects, and rekha's audit measured per-point
`sd_point_new` as the dominant remaining emit cost (a 21-point line path at ~704 ns). Inline (x, y)
storage in the points array would take the ASCII set to 58,672 B and remove one allocation per point.
That is an ABI change to `SdPath` and a sadish design decision; the numbers are here so it can be made.
