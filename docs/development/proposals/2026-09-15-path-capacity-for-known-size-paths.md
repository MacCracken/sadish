# Capacity-sized paths: `sd_path_new_cap(n_verbs, n_points)` for callers that know the path's size

**Status:** 🟡 **OPEN — SHIPPED in 0.7.1 as `sd_path_new_cap(n_verbs, n_points)`, with a deviation from
this proposal's first bullet.** The call is public, bounded at both ends, in `dist/`, and gated by
checks that fail under ten separate reversions — a real capability, not a rubber stamp. What it is not
is *"the verb and point arrays at exactly the given capacity"*: one `SD_PATH_CAP_OFFSET` field governs
both arrays, `n_points` reaches the allocator only through `max()`, and this proposal's own target of
78,656 B is missed by 5,840 B (7.4 %). That is disclosed in "⚠ ONE capacity, not two" below; it was
not disclosed in this line, which read "🟢 **CLOSED — SHIPPED**" until the 2026-09-15 re-audit against
HEAD `f722e8c`. See **Still open** at the foot. Was 🟡 OPEN — a capability request.
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

## MEASURED (rekha 0.3.11 on sadish 0.5.5 — sadish 0.6.0's `sd_path_new` was the same code)

⚠ "the same code" was true through 0.6.0 and is stale after 0.7.1, which made `sd_path_new` a
one-liner over the shared `_sd_path_alloc` with three 0-checks. Re-derived 2026-09-15: every byte
figure below still checks out arithmetically and against checks #134 / #137 / #138.

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
  ⚠ That pin is in rekha's repo and is not verifiable here; sadish's own `programs/alloc_test.cyr`
  contains no 4,192 / 4,144 figure. The IN-TREE pin is `programs/flatten_bound_test.cyr:811`
  (check #122), which holds.
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
48 B record) and the 2 GiB array behind it.

⛔ **The two sentences that used to end that paragraph were both wrong, and the re-audit measured
each.** They read: *"AT it, it makes one (the 48 B record) and `lib/alloc.cyr` refuses the 2 GiB array
behind it. Both answer 0, which is why raising the constant to 2^29 passed every suite until
`programs/flatten_bound_test.cyr`'s group N started counting the calls."*

- **AT the ceiling nothing is refused.** `lib/alloc.cyr:225` refuses only `size > ALLOC_MAX`, and
  `ALLOC_MAX` (`lib/alloc.cyr:174`) is `0x80000000` — EXACTLY `2^28 * 8`. MEASURED 2026-09-15 under
  the stock allocator: `sd_path_new_cap(268435456, 0)` returns a **live path** with `cap` 268435456,
  `sd_path_moveto` returns `SADISH_OK`, and the verb count reads 1. It does not answer 0. Checks #299
  / #300 run under `hook_grant` with `g_grant = 0` — an allocator that refuses everything — so they
  never reach `lib/alloc.cyr`, and `programs/flatten_bound_test.cyr:1553` ("sadish asks; alloc says
  no") carries the same error. `src/path.cyr:39-40` states it correctly ("the largest array the stock
  allocator can EVER serve") and so contradicts this file. ⚠ `sd_path_new_cap(2^28, 0)` really does
  hand back a live path backed by 4 GiB of lazily-committed mapping. That is defensible — the ceiling
  is a WRAP GUARD, not a size policy — but it is the opposite of what this proposal said happens.
- **Raising the constant to 2^29 does NOT pass every suite.** Check #222
  (`programs/flatten_bound_test.cyr:1192`, group **M1**, which predates group N) pins
  `SD_PATH_CAP_MAX == 268435456` literally. MEASURED: constant raised to 2^29 → #222 alone fails. What
  group N catches is a WIDENED COMPARISON with the constant intact (`cap > SD_PATH_CAP_MAX * 2`, which
  is what `programs/flatten_bound_test.cyr:1539` says was mutation-proved) → #296 and #298 alone. Two
  different mutations, conflated into one sentence.

Same `SdPath` layout, same doubling growth on overflow, and `sd_path_new()` unchanged in capacity,
layout and cost — **4,144 B in 3 allocations** (checks #121-#123). ⚠ Not "byte-for-byte as it was",
which this line used to claim: 0.7.0's `sd_path_new` had an inline body with one 0-check, HEAD's is
`return _sd_path_alloc(SD_PATH_CAP);` (`src/path.cyr:92-94`) with three — as "Also shipped" below says
in as many words. ⚠ Growth is not byte-for-byte either: `sd_path_grow` gained a refusal at
`cap > SD_PATH_CAP_MAX / 2` (`src/path.cyr:142`) that 0.7.0 did not have. Unreachable from sadish's own
constructors, so harmless, but "same doubling growth on overflow" does not mention it.

### ⚠ ONE capacity, not two — and what that costs

`SdPath` has a single `SD_PATH_CAP_OFFSET` field governing BOTH arrays, so `n_verbs` and `n_points`
**collapse to their max** and both arrays are opened at it. A glyph therefore pays about 2× more
verb slots than it uses. The alternative — a second capacity field — is an ABI change:
`SD_PATH_RESERVED_OFFSET` (+40) could carry it at the same 48 B size, but the meaning of that word
is published, `sd_path_grow` would have to double the two arrays independently (moving figures
already asserted in the in-tree allocation gates and every consumer's), and **crab
vendors its own copy of `dist/sadish.cyr`**. ⚠ This used to name `programs/grow_test.cyr` as the
suite holding those figures. It does not: `grow_test` asserts verb/point COUNTS and first/last point
coordinates (`programs/grow_test.cyr:29-38`), none of which independent doubling would move. The
figures that would actually move are `programs/flatten_bound_test.cyr:867` and
`programs/stroke_style_test.cyr:694`. ⇒ 0.7.1 takes the capability without the ABI risk. The
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
**98.4 % of the saving** — 349,152 B saved of a possible 354,992 B. Per glyph: 4,565 B → **889 B**
(target 828 B). Not one of the 95 paths grows (asserted): an exact estimate never doubles.
⚠ This line used to say "93 % of the saving", which is `5.13 / 5.51` — the ratio of the two reduction
FACTORS, not of the bytes saved. Corrected 2026-09-15.

⚠ The 95 "ASCII" paths in the gate are SYNTHETIC (`programs/flatten_bound_test.cyr:364-377` — moveto +
a linetos + b quadtos + close, hand-tuned so the totals land on 1,768 / 2,498 / 61). Checks
#131-#133 are therefore a self-consistency pin, not an independent re-measurement of LiberationSans;
the paragraph above discloses this ("synthesises 95 paths of that shape"). The figures that matter
(#134 / #135) are real byte counts through the allocation seam.

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
exact-capacity bytes are `SdPoint` objects. Re-verified 2026-09-15: `SdPath` still stores pointers
(`src/path.cyr:50`) and `sd_path_push_point` still takes one (`src/path.cyr:172`); check #132 pins the
39,968. This proposal labels it "not requested here", so it is declared residue rather than a broken
promise — but on this file's own numbers it is the largest item left (58,672 B vs 78,656 B).

---

## Still open (re-audited 2026-09-15 against 0.7.1, HEAD `f722e8c`)

1. **"allocate the verb and point arrays at exactly the given capacity"** — not what shipped, and the
   reason this stays in `proposals/` rather than `proposals/archived/`. One capacity field, `n_points`
   reaching the allocator only through `max()`, 84,496 B against the 78,656 B computed above. ⚠ The
   ask also conflicts with the same bullet's "same `SdPath` layout"; sadish chose the layout
   deliberately, and "⚠ ONE capacity, not two" gives the reasoning. Closing this means the ABI change.
2. **No in-tree adopter.** `programs/flatten_bound_test.cyr` is the ONLY file that calls
   `sd_path_new_cap`. sadish's own two known-size path sites still call `sd_path_new`:
   `src/stroke.cyr:133` (`sd_stroke_seg`, 5 verbs / 4 points) and `src/stroke.cyr:145`
   (`sd_stroke_disc`, 17 verbs / 16 points). MEASURED: one round stroke of a 4-vertex rect costs
   **34,432 B in 104 allocations** today (pinned at `programs/stroke_style_test.cyr:694`); with those
   two sites on `sd_path_new_cap(5, 4)` and `sd_path_new_cap(17, 16)` it is **3,264 B in the SAME 104
   allocations** — 10.5× less, every other suite still green, only that pinned figure moving. Not
   something this proposal asks for, but it is exactly the arena cost it was opened about.
   ⚠ Those same two lines are the unchecked `sd_path_new()` results in
   `issues/2026-09-15-path-construction-stores-through-a-refused-allocation.md`; one edit closes both.
3. **rekha adoption** — *"rekha updates that test when it adopts the new call"* is unverifiable from
   this repo, and nothing here tracks it.
