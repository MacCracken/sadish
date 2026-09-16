# Capacity-sized paths: `sd_path_new_cap(n_verbs, n_points)` for callers that know the path's size

**Status:** 🟢 **CLOSED — SHIPPED in sadish 0.7.1** as `sd_path_new_cap(n_verbs, n_points)` (see the
closing section at the foot of this file). Was 🟡 OPEN — a capability request.
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

---

## SHIPPED — sadish 0.7.1 (`src/path.cyr`, `programs/flatten_bound_test.cyr`)

```cyr
fn sd_path_new_cap(n_verbs, n_points): i64    # 0 on a refused allocation
var SD_PATH_CAP_MIN = 8;                      # the floor a 0/negative estimate is raised to
var SD_PATH_CAP_MAX = 268435456;              # 2^28 entries — ABOVE it the call is REFUSED
```

### ⛔ Both ends of the estimate are bounded

A 0 or negative estimate is raised to `SD_PATH_CAP_MIN`; a capacity above `SD_PATH_CAP_MAX` returns
**0**, and that end is a refusal rather than a clamp. Both arrays are `cap * 8` bytes and **that
multiply wraps**: MEASURED before the ceiling, `sd_path_new_cap(2^61 + 1, 0)` returned a live path
whose verb and point blocks were **8 B apart** — both `sd_alloc` calls succeeded, so both 0-checks
passed — carrying a stored capacity of 2.3e18, and the second push wrote off the end of the array.
2^28 entries is 2 GiB per array, `lib/alloc.cyr`'s `ALLOC_MAX`, so every capacity above it was
already refused by the allocator; the ceiling changes only the wrap windows. ⚠ A capacity a caller
computes from a file's own counts is exactly the argument that reaches 2^61.
⚠ The ceiling is SADISH's refusal, not the allocator's, and only the ALLOCATION COUNT tells them
apart: over the ceiling, `sd_path_new_cap` makes ZERO `sd_alloc` calls; AT it, it makes one (the
48 B record) and `lib/alloc.cyr` refuses the 2 GiB array behind it. Both answer 0, which is why
raising the constant to 2^29 passed every suite until `programs/flatten_bound_test.cyr`'s group N
started counting the calls.

Same `SdPath` layout, same doubling growth on overflow, `sd_path_new()` byte-for-byte as it was.

### ⚠ ONE capacity, not two — and what that costs

`SdPath` has a single `SD_PATH_CAP_OFFSET` field governing BOTH arrays, so `n_verbs` and `n_points`
**collapse to their max** and both arrays are opened at it. A glyph therefore pays about 2× more
verb slots than it uses. The alternative — a second capacity field — is an ABI change:
`SD_PATH_RESERVED_OFFSET` (+40) could carry it at the same 48 B size, but the meaning of that word
is published, `sd_path_grow` would have to double the two arrays independently (moving figures
already asserted in `programs/grow_test.cyr` and every consumer's allocation gate), and **crab
vendors its own copy of `dist/sadish.cyr`**. ⇒ 0.7.1 takes the capability without the ABI risk. The
two-argument SIGNATURE is the one a split-capacity record would keep, so no rekha call site changes
if sadish ever separates them.

### MEASURED on this tree — the proposal's own set, both ways

Every printable ASCII glyph of LiberationSans through `rekha_outline_to_sdpath` at 32 px (rekha
0.3.11 against this worktree's dist): **1,768 verbs, 2,498 points, largest glyph 61 verbs** — the
proposal's numbers exactly. `programs/flatten_bound_test.cyr` synthesises 95 paths of that shape and
builds them through `sd_path_moveto/lineto/quadto/close` on a counting seam hook:

| | bytes | vs today |
|---|---:|---:|
| `sd_path_new` + pushes (today) | **433,648** | — |
| `sd_path_new_cap`, ONE capacity (0.7.1) | **84,496** | **5.13× less** |
| the same at SEPARATE capacities (this proposal's target) | 78,656 | 5.51× less |

⇒ One capacity costs **5,840 B (7.4 %)** over the two-capacity target on the ASCII set, and captures
93 % of the saving. Per glyph: 4,565 B → **889 B** (target 828 B). Not one of the 95 paths grows
(asserted): an exact estimate never doubles.

A small path is unchanged: `sd_path_new` + moveto + 2 linetos + close is still **4,192 B in 6
allocations** (rekha's `alloc_test` pin holds). The same path through `sd_path_new_cap(4, 3)` is
**224 B in the same 6 allocations**.

### Not a cap

An underestimate stays correct — `sd_path_grow` doubles exactly as it does for `sd_path_new`. The
suite builds the largest glyph at `sd_path_new_cap(1, 1)` and at its exact size and compares the two
paths verb for verb and point for point.

### Also shipped, from the companion issue

Every allocation in `sd_path_new` / `sd_path_new_cap` (record, verbs, points) is checked, and the
builders now RESERVE capacity, then allocate their `SdPoint`s, then push — so a refusal returns
`SADISH_ERR_OOM` with the path exactly as it was, never a verb without its points. See
`issues/2026-09-15-path-construction-stores-through-a-refused-allocation.md`.

### Not done (still open, deliberately)

Inline `(x, y)` storage instead of `SdPoint` pointers — the "worth measuring alongside" section
above. It is an ABI change and was out of scope for 0.7.1. Its number stands: 39,968 of the 78,656
exact-capacity bytes are `SdPoint` objects.
