# Capacity-sized paths: `sd_path_new_cap(n_verbs, n_points)` for callers that know the path's size

**Status:** 🟡 **STILL OPEN — items 1 and 2 are closed; item 3 is a REKHA ask this repo cannot
verify.** 0.7.2 gave the verb and point arrays separate capacities (`SD_PATH_CAP_OFFSET` = verbs,
`SD_PATH_PCAP_OFFSET` = points, in the 48 B record's former `reserved` word), so
`sd_path_new_cap(n_verbs, n_points)` opens each array at exactly its own requested capacity and each
doubles alone: **MEASURED 78,656 B on this proposal's own ASCII set, its target to the byte**
(`programs/path_cap_test.cyr` #160, 170 checks, 24 single mutations each killed) — item 1.
0.8.0 puts sadish's own two known-size sites on that call: `sd_stroke_seg` at
`sd_path_new_cap(5, 4)` and `sd_stroke_disc` at `sd_path_new_cap(17, 16)`, so the proposal at last
has an IN-TREE ADOPTER — **MEASURED, the closed 8x8 rect stroke 34,432 B → 3,232 B in the SAME 104
`sd_alloc` calls** (`programs/path_cap_test.cyr` group N, 82 of the suite's 252 checks;
`programs/stroke_style_test.cyr` #13) — item 2.
0.9.0 takes the item this file called *"worth measuring alongside"* and left as residue — **inline
`(x, y)` storage instead of `SdPoint` pointers** — and hits its number: **MEASURED 58,672 B on this
same ASCII set, the figure the section below computed in 0.7.1, in 285 `sd_alloc` calls against
2,783** (`programs/path_cap_test.cyr` #160 / #253, `programs/inline_points_test.cyr`). See "Worth
measuring alongside" below, rewritten as the record of what shipped.
⛔ **This filing STILL does not archive:** item 3,
*"rekha updates that test when it adopts the new call"*, is a promise made on rekha's behalf and
nothing in this repo can witness it — and 0.9.0 makes the ask LARGER rather than smaller, because
rekha now has to port 5 test programs' point-array reads as well (the filing sadish is sending). Was 🟡 STILL OPEN — the CAPABILITY complete as of 0.7.2 and the
ADOPTION asks not; 🟡 OPEN — SHIPPED in 0.7.1 with a deviation from this proposal's first bullet
before that; and 🟡 OPEN — a capability request before that. It read "🟢 **CLOSED — SHIPPED**" until
the 2026-09-15 re-audit against HEAD `f722e8c`.
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
  `issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md`).

## Worth measuring alongside (not requested here) — ⭐ SHIPPED, sadish 0.9.0

⚠ **The paragraph this section used to be is kept verbatim below; everything after it is what the
measurement turned into.** It read:

> `SdPath` stores POINTERS to 16 B `SdPoint`s, so every point costs 24 B and one `sd_alloc` call — 39,968
> of the 78,656 exact-capacity bytes above are `SdPoint` objects, and rekha's audit measured per-point
> `sd_point_new` as the dominant remaining emit cost (a 21-point line path at ~704 ns). Inline (x, y)
> storage in the points array would take the ASCII set to 58,672 B and remove one allocation per point.
> That is an ABI change to `SdPath` and a sadish design decision; the numbers are here so it can be made.

The decision was made and 0.9.0 is that change, shipped alone. `SdPath`'s points array holds the
COORDINATES, 16 B a slot (x at +0, y at +8), at the same `SD_PATH_POINTS_OFFSET` = +16 in the same
48 B record. Read them with **`sd_path_point_x(path, i)` / `sd_path_point_y(path, i)`** — new, and
the published way; `sd_path_verb_at(path, i)` joins them for the verb stream, which did not change
shape.

### MEASURED on this tree — this proposal's own set, all four ways

| | bytes | `sd_alloc` calls | per glyph |
|---|---:|---:|---:|
| `sd_path_new` + pushes, 0.8.0 | 433,648 | 2,783 | 4,564 B |
| `sd_path_new` + pushes, 0.9.0 | **393,680** | **285** | 4,144 B |
| `sd_path_new_cap`, two capacities, 0.8.0 | 78,656 | 2,783 | 828 B |
| `sd_path_new_cap`, two capacities, 0.9.0 | **58,672** | **285** | **618 B** |

⇒ **58,672 B, this section's own figure, to the byte** = 95·48 + 8·1,768 + 16·2,498, and the
19,984 B it saves against 0.8.0 is exactly the pointer array it no longer needs (8 B × 2,498).
⇒ **285 calls = 3 × 95.** A path costs three `sd_alloc` requests — record, verb array, point array —
whatever its point count, where 0.8.0 paid one more per point. That is the section's "remove one
allocation per point", counted. Gated by `programs/path_cap_test.cyr` group L (#158-#165) and
group O (#253-#258).

### MEASURED — the per-point emit cost this section cites

The ~704 ns is rekha's figure on rekha's tree and cannot be reproduced from here; what CAN be is the
same SHAPE, timed on this machine, before and after. `sd_path_new` + `moveto` + 20 `lineto`s (21
points), built and dropped, median of 9 rounds of 20,000 builds, `CLOCK_MONOTONIC_RAW` via
`lib/bench.cyr`'s `now_ns()`, loop overhead measured separately at 3 ns:

| | 0.8.0 | 0.9.0 | |
|---|---:|---:|---:|
| 21 points via `sd_path_new` | 1,041 ns | **841 ns** | 1.24× |
| 21 points via `sd_path_new_cap(21, 21)` | 830 ns | **563 ns** | 1.47× |
| 200 points via `sd_path_new_cap(200, 200)` | 7,500 ns | **4,882 ns** | 1.54× |

⇒ 27 ns a point at exact capacity, against 40. ⚠ Wall clock on one idle machine, not a benchmark
harness: the figures repeat to ±2 % across runs and the ratio is the claim, not the absolute.

### What this cost, said plainly

- **`sd_path_new`'s default point capacity halved, 256 slots → `SD_PATH_PCAP` = 128**, so its first
  allocation is the same 48 + 2,048 + 2,048 = **4,144 B in 3 calls** it has been since 0.4.0. At 256
  inline slots it would have been 6,192 B — and dhancha's 512 KiB text arena CHAINS ANOTHER CHUNK on
  a 60-glyph run at that size (MEASURED, its own `text_arena_test`). The price is one doubling for a
  path of 129-256 points, where 0.8.0 needed none.
- **`SD_PATH_CAP_MAX` halved, 2^28 → 2^27**, re-derived for the 16 B slot: 2^27 × 16 = 2 GiB =
  `lib/alloc.cyr`'s `ALLOC_MAX`. One constant bounds both arrays and takes the tighter one, so a
  VERB capacity in (2^27, 2^28] is now refused that 0.8.0 served.
- **`sd_path_flatten` allocates an `SdPoint` per emitted point**, where it used to hand the path's
  own record through for a straight verb's anchor and a curve's endpoint: +16 B and one call per
  ANCHOR and per CURVE VERB. The polyline still holds `SdPoint` pointers — a second ABI break in one
  release is how consumers lose trust — so this is what keeping it costs. MEASURED: a 21-point line
  path flattens in 24 calls against 3, while BUILDING it fell from 24 to 3; and the CHANGELOG's own
  one-quad fill (`moveto` (0,0), `quadto` ctrl (8,0) → (8,8), `lineto` back, closed, warm scratch)
  costs **64 B in 4 seam allocations against 0.8.0's 48 B in 3** — the 3 emitted mid-points as
  before, plus the curve's own end point, which used to be the path's record handed through.
- **Every walk's "no moveto yet" test became a flag.** `cur != 0` was free while the current point
  was a pointer; (0, 0) is a real point, so a path at the ORIGIN would otherwise draw nothing.
  `programs/inline_points_test.cyr` groups E and I are that gate, and mutation testing is what
  showed group E's `> 0` checks were too weak to see it.
- ⛔ **The growth ceiling buys HALF the run and dash points under the same bytes.**
  `sd_grow_limit_set` is denominated in BYTES (8 MiB by default); the stroker's run and the dash
  piece buffer are 16 B a slot now, so they top out at **524,288 points, where 0.8.0 reached
  1,048,576** — MEASURED on both trees. Past it a stroke sets `_sd_run_trunc`, drops the tail and
  returns `SADISH_ERR_OOM`, so a single subpath longer than that truncates where 0.8.0 stroked it
  whole (MEASURED with the ceiling lowered to 262,144 B and a 20,000-point styled stroke: 0.8.0
  `SADISH_OK` and 526,870 coverage units, 0.9.0 `SADISH_ERR_OOM` and 434,050). The fill's edge and
  crossing ceilings did NOT move — their elements did not change size — so the hostile-path defence
  is exactly where 0.7.0 put it. ⇒ The remedy is one call, `sd_grow_limit_set(16777216)`: the 0.8.0
  point ceiling at twice the bytes. The default is not raised, because the knob's contract is
  bounded MEMORY and doubling it would double the EDGE ceiling too. Swept 2026-09-16: no repo in the
  ecosystem calls `sd_grow_limit_set` at all. Pinned by `programs/grow_edges_test.cyr` group R
  (#217-#230) so neither the slot nor the default can move again silently — and R2 pins the remedy
  itself, because a documented one-call fix that stops working is worse than no fix.
  ⭐ **And the remedy costs no real memory.** MEASURED on both trees at 8,192 / 16,384 /
  32,768-point subpaths, where the run's capacity is exactly its occupancy, a stroked point costs
  **40 B of path + run on either release** — 0.8.0 spends 8 of them on the run and 32 on the path
  (8 B verb + 8 B pointer + 16 B `SdPoint`), 0.9.0 spends 16 and 24 (8 B verb + 16 B inline slot).
  What moved is the ceiling's SHARE of an unchanged total. So at their truncation points: 0.8.0
  holds 1,048,576 × 40 = **41,943,040 B**, 0.9.0 holds 524,288 × 40 = **20,971,520 B**, and 0.9.0
  under `sd_grow_limit_set(16777216)` holds 1,048,576 × 40 = **41,943,040 B** — 0.8.0's longest
  subpath in 0.8.0's own footprint, to the byte. ⇒ The default refuses at half the total memory it
  used to, and the knob restores the old length without asking for a byte more than 0.8.0 spent.
  ⚠ This is the one behaviour change in a release billed as one change, and it is the maintainer's
  to accept or reverse: mutation MF7 (`SD_GROW_LIMIT_DEFAULT` → 16777216) is the lever, and group
  R's #218/#222/#224 are the checks it moves.
- **The stroker's run costs 393,216 B more of global heap over a long sweep**, and the caller gets
  most of it back: re-running the 100..25,600-segment sweep behind `src/stroke.cyr`'s batch note,
  the run and its flags went 442,368 B → 835,584 B while the same walks' PATHS fell 408,800 B on the
  seam — MEASURED **+49,952 B net** across the nine walks.

### What is deliberately NOT done

`SdPoint`, `sd_point_new`, `sd_matrix_apply` and `SdPolyline` are untouched — the polyline is a
different record with its own 0.8.0 verdict word, and one ABI break per release is the whole
sequencing argument. The curve recursion still materialises the points it emits.

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
  no") carries the same error. `SD_PATH_CAP_MAX`'s own comment block (`src/path.cyr:39-40` at 0.7.1,
  `:40-41` at 0.7.2) states it correctly ("the largest array the stock allocator can EVER serve") and
  so contradicts this file. ⚠ `sd_path_new_cap(2^28, 0)` really does hand back a live path backed by
  4 GiB of lazily-committed mapping. That is defensible — the ceiling
  is a WRAP GUARD, not a size policy — but it is the opposite of what this proposal said happens.
- **Raising the constant to 2^29 does NOT pass every suite.** Check #222
  (`programs/flatten_bound_test.cyr:1192`, group **M1**, which predates group N) pins
  `SD_PATH_CAP_MAX == 268435456` literally. MEASURED: constant raised to 2^29 → #222 alone fails. What
  group N catches is a WIDENED COMPARISON with the constant intact (`cap > SD_PATH_CAP_MAX * 2`, which
  is what `programs/flatten_bound_test.cyr:1539` says was mutation-proved) → #296 and #298 alone. Two
  different mutations, conflated into one sentence.

Same `SdPath` layout, same doubling growth on overflow, and `sd_path_new()` unchanged in capacity,
layout and cost — **4,144 B in 3 allocations** (checks #121-#123). ⚠ Not "byte-for-byte as it was",
which this line used to claim: 0.7.0's `sd_path_new` had an inline body with one 0-check, 0.7.1's is
`return _sd_path_alloc(SD_PATH_CAP);` (`sd_path_new`, `src/path.cyr:92-94` at 0.7.1) with three — as
"Also shipped" below says in as many words. ⚠ Growth is not byte-for-byte either: `sd_path_grow`
gained a refusal at `cap > SD_PATH_CAP_MAX / 2` (`src/path.cyr:142` at 0.7.1, `:197` at 0.7.2) that
0.7.0 did not have. Unreachable from sadish's own constructors, so harmless, but "same doubling
growth on overflow" does not mention it.

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
`issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md`.

### Not done (still open, deliberately) — ⭐ CLOSED IN 0.9.0

⚠ **0.7.1's words are kept; 0.9.0 is what answered them.** They read: *"Inline `(x, y)` storage
instead of `SdPoint` pointers — the "worth measuring alongside" section above. It is an ABI change
and was out of scope for 0.7.1. Its number stands: 39,968 of the 78,656 exact-capacity bytes are
`SdPoint` objects … on this file's own numbers it is the largest item left (58,672 B vs 78,656 B)."*
That is exactly what shipped, at exactly that number — see "Worth measuring alongside" above.
`sd_path_push_point` now takes two coordinates rather than an `SdPoint`, and the layout comment says
so.

---

## CLOSED — item 1, sadish 0.7.2 (`src/path.cyr`, `programs/path_cap_test.cyr`)

⚠ **Everything above this line is 0.7.1's record and is now history.** The "⚠ ONE capacity, not two"
section and the three-row MEASURED table in it describe the release that is being superseded; they
are left verbatim because the measurement outlives the decision. What follows is what 0.7.2 does.

```cyr
fn sd_path_new_cap(n_verbs, n_points): i64   # each array at exactly its own capacity
fn sd_path_verb_cap(path): i64               # new — what the verb array holds before it doubles
fn sd_path_point_cap(path): i64              # new — what the point array holds
fn sd_path_grow(path, which): i64            # was sd_path_grow(path); doubles ONE array
var SD_PATH_PCAP_OFFSET = 40;                # was SD_PATH_RESERVED_OFFSET (removed, not aliased)
var SD_PATH_GROW_VERBS  = 0;
var SD_PATH_GROW_POINTS = 1;
```

⭐ **The record is still 48 B.** The point capacity went into the `reserved` word at +40 — written 0
by every 0.7.1 constructor and read by nothing. Grepped before it was taken, and **re-run 2026-09-16
over the complete set of repos**: no `SD_PATH_*` constant beyond `VERBS_OFFSET` / `POINTS_OFFSET` /
`NVERBS_OFFSET` / `NPOINTS_OFFSET` is referenced outside this repo (rekha reads the two array
pointers in five test programs — 8 `SD_PATH_VERBS_OFFSET` and 6 `SD_PATH_POINTS_OFFSET`, nothing
else; dhancha mentions `sd_path_new`'s cost only in comments; agnos's `refagree` calls
`sd_path_new`), **no repo outside this one calls `sd_path_grow`** — whose arity changed — and
nothing anywhere hand-builds an `SdPath`.
⚠ **TWO repos vendor `dist/sadish.cyr` wholesale, not one**: `crab/lib/sadish.cyr` **and
`puka/lib/sadish.cyr`** (line 925, still carrying 0.7.1's `SD_PATH_RESERVED_OFFSET`). This paragraph
named only crab until 2026-09-16. The conclusion does not change — `puka/src`, `puka/programs` and
`puka/tests` name no `SD_PATH_*` constant and call no `sd_path_*` function, and `puka/cyrius.cyml`
declares no sadish dep, so that copy is inert — but the next person re-running this sweep needs the
complete set of places a stale copy lives.
⚠ The one shape that changes meaning is exactly that: a record a consumer
built to the 0.7.1 layout carries 0 at +40, and a 0 point capacity is now REFUSED
(`SADISH_ERR_OOM`, zero allocation attempts) rather than written through — the safe direction,
pinned by `programs/path_cap_test.cyr` group K.

### MEASURED on this tree — the proposal's own set, all three ways

95 ASCII-shaped paths (1,768 verbs, 2,498 points, largest glyph 61 verbs) built through
`sd_path_moveto/lineto/quadto/close` on the allocation seam (`programs/path_cap_test.cyr` group L;
the 0.7.1 row is EMULATED by calling `sd_path_new_cap(m, m)` with `m = max(v, p)`, the argument pair
0.7.1's own `max()` produced):

| | bytes | vs `sd_path_new` | per glyph |
|---|---:|---:|---:|
| `sd_path_new` + pushes | **433,648** | — | 4,564 B |
| `sd_path_new_cap`, ONE capacity (0.7.1) | **84,496** | 5.13× less | 889 B |
| `sd_path_new_cap`, two capacities (0.7.2) | **78,656** | **5.51× less** | **828 B** |

⇒ **78,656 B — this proposal's target to the byte**, = 95·48 + 8·1,768 + 8·2,498 + 16·2,498. The
5,840 B (6.9 %) one capacity left on the table is gone, and the whole 354,992 B saving the proposal
computed is now captured. Not one of the 95 paths grows in either array (asserted). ⚠ 39,968 B of
the 78,656 — 51 % — are still the `SdPoint` objects; inline (x, y) storage remains the largest item
left and remains out of scope (see "Not done (still open, deliberately)" above).

### What else moved, and what did not

- **Growth is per array.** `sd_path_grow(path, which)` doubles the array that overflowed and leaves
  the other one alone — one allocation where 0.7.1 made two. MEASURED: a path at `(8 verbs, 64
  points)` full at 8/8 pays **144 B in 2 allocations** for the lineto that overflows its verbs (one
  128 B block + the `SdPoint`), and its point capacity stays 64. Even `sd_path_new` benefits: the
  257th verb of a default path costs **one 4,096 B block**, not two.
- **`sd_path_new` is unchanged to the byte** — 4,144 B in 3 allocations, 4,192 B for rekha's small
  path, the three requests still 48 / 2,048 / 2,048 in that order.
- ⛔ **Rendering does not move.** agnos `refagree`: **BYTE-IDENTICAL on all 200 paths**. 28 suites
  green; rekha (23) and dhancha (18) pass against this `dist/`.
- The overflow guard holds for **both** arrays: either capacity above `SD_PATH_CAP_MAX` is refused
  before any `cap * 8`, with **zero** allocation attempts — including an over-ceiling POINT capacity,
  which is tested before the verb array is allocated. `sd_path_grow` still refuses past
  `SD_PATH_CAP_MAX / 2`, now per array.
- **Seven** assertions in `programs/flatten_bound_test.cyr` moved with the behaviour (#112, #115,
  #128, #130, #135, **#136**, #217) plus group J3's fixture; they are listed in that file and in the
  0.7.2 release notes. #136 is the one that is easy to miss and is counted here deliberately: its
  `igrew` loop went from ONE comparison (`load64(gp + SD_PATH_CAP_OFFSET) != glyph_npoints(ii)`) to
  TWO — the verb capacity AND the point capacity must each equal that glyph's exact count — so what
  the assertion tests changed even though its expected value stayed 0. ⚠ This line read "Six" and
  omitted #136 until 2026-09-16; a status line that undercounts its own edits is the failure mode
  this release exists to stop repeating.
- **The verb array is read back after a growth now, in the only three places where it can be read
  back at all.** ⛔ The contents checks in `programs/path_cap_test.cyr` all walked paths with EQUAL
  verb and point counts, and a verb growth that took its copy length from the POINT count then
  copies exactly the right number of bytes. MEASURED with that one-word mutation (`n_off =
  SD_PATH_NPOINTS_OFFSET` in `sd_path_grow`): 26 of the 28 suites passed — all 156 checks of
  `path_cap_test` and all 304 of `flatten_bound_test` among them — and only `programs/area_test.cyr`
  (#131: 6,205 of 7,800 coverage units) and `programs/grow_edges_test.cyr` (SIGSEGV) caught it.
  Groups **E3** (9 closes, 0 points), **E4** (moveto + 8 closes, 1 point — a PARTIAL copy, not an
  empty one) and **G** (257 closes, 0 points) now read every verb back; each kills that mutation on
  its own (#68 got 8 bad verbs, #76 got 7, #114 got 256). ⚠ Why only those three: a lost point slot
  is a null `SdPoint` the next walk faults on, but a lost verb tag reads back as a valid
  `SD_VERB_MOVETO` — so the verb array is the quiet half, and only a path whose verbs outnumber its
  points can show it.

---

## CLOSED — item 2, sadish 0.8.0 (`src/stroke.cyr`, `programs/path_cap_test.cyr` group N)

⚠ **Everything above this line is 0.7.1's and 0.7.2's record.** What follows is the ADOPTION — the
item that said `sd_path_new_cap` had no caller in `src/` at all.

```cyr
fn sd_stroke_seg(cv, ax, ay, bx, by, hw): i64 {
    var rp = sd_path_new_cap(5, 4);        # was sd_path_new()
fn sd_stroke_disc(cv, cx, cy, hw): i64 {
    var dp = sd_path_new_cap(17, 16);      # was sd_path_new()
```

**Two lines. Nothing else in `src/` changed** beyond the comments that carried the old figures
(`sd_stroke_seg`'s and `sd_stroke_disc`'s headers, this file's header block, and
`_sd_scratch_init`'s in `src/raster.cyr`).

### Where the counts come from — DERIVED, not estimated

- `sd_stroke_seg` calls `sd_path_moveto` once, `sd_path_lineto` three times and `sd_path_close`
  once: **5 verbs**; moveto and each lineto carry one point, close carries none: **4 points**. Both
  are under `SD_PATH_CAP_MIN` = 8, so each array is opened at 8.
- `sd_stroke_disc`'s `while (k < 16)` emits one moveto and 15 linetos, then `sd_path_close` adds the
  17th verb: **17 verbs, 16 points**, and both arrays are EXACTLY full when the disc is built.
- ⛔ **A SWEEP OF `src/` FOUND NO THIRD SITE.** `sd_path_new` is called nowhere else in `src/`
  (grep, 2026-09-16); `src/dash.cyr` builds no `SdPath` at all — its piece buffer, curve flags and
  four cut-point records come from the GLOBAL `alloc` in `_sd_dash_init`, not from a path — and the
  styled stroker's discs are edges in `_sd_sb_disc`, not paths. The two sites this proposal named
  are the whole set.

### MEASURED on this tree — the same calls before and after, on the allocation seam

| | 0.7.2 | 0.8.0 | | `sd_alloc` calls |
|---|---:|---:|---:|---:|
| `sd_stroke_seg`, one call | 4,208 B | **240 B** | 17.5× less | **7 → 7** |
| `sd_stroke_disc`, one call | 4,400 B | **568 B** | 7.75× less | **19 → 19** |
| closed 8x8 rect stroke (this proposal's own quoted case) | 34,432 B | **3,232 B** | 10.65× less | **104 → 104** |
| round stroke of a cubic (8 rects, 9 discs, 7 mid-points) | 73,376 B | **7,144 B** | 10.27× less | **234 → 234** |
| 54-glyph label, round-stroked (a dhancha-shaped text draw) | 16,357,904 B | **1,557,176 B** | 10.50× less | **50,593 → 50,593** |

⇒ 240 = 48 + 64 + 64 + 4·16; 568 = 48 + 136 + 128 + 16·16; 3,232 = 4·240 + 4·568. The item's own
estimate was **3,264 B**, computed under 0.7.1's single capacity; re-derived here as this file asked
("re-derive it when the adoption is made rather than quoting it"), it is **3,232 B** — 4 × 8 B lower,
one 8 B verb slot per disc path, exactly the correction the item predicted.
⚠ **The 54-glyph label is the figure that answers the arena filing this proposal was opened about**:
14,800,728 B off one label's arena, per draw. It is a synthetic corpus (this file's own ASCII-shaped
paths, round-stroked at width 1), not a rekha/dhancha measurement — the shape is honest, the font is
not real.

### ⛔ THE ALLOCATION COUNT IS THE SECOND HALF OF EVERY ROW, AND IT IS THE PROOF OF NO GROWTH

`sd_path_new_cap` makes the same three requests `sd_path_new` did — record, verb array, point array
— for smaller blocks. So an unchanged count means **nothing was batched away, no piece was dropped,
and NEITHER ARRAY EVER DOUBLED**: a growth is an extra `sd_alloc` and would show. Three independent
gates say it:

- `programs/path_cap_test.cyr` group N reads the capacities and the counts **off the record the
  allocation hook handed the real site** (not off a rebuild of its geometry): `sd_stroke_seg`'s path
  is 5 verbs in an 8-slot array and 4 points in an 8-slot array (#180-#185); `sd_stroke_disc`'s is
  **17 of 17 and 16 of 16** (#193-#196) — count EQUALS capacity in both arrays.
- `programs/stroke_oom_test.cyr` #14 pins the round stroker's own allocation count for a 2-point
  line at `7 + 19 + 19` = 45, and it is **unchanged**. MEASURED as mutations: `(16, 16)` on the disc
  reads 47 there, and so do `(17, 15)` and `(16, 17)`.
- Group N #197-#202 measures what the off-by-one costs: the same 17-verb / 16-point shape opened at
  `(16, 16)` is **816 B in 20 allocations** against 568 B in 19 — **248 B and one memcpy** for one
  slot. ⚠ Still far under the 4,400 B the 256-slot default cost, so a rounded-down capacity is not
  "worse than before"; 248 B is what deriving the count off the code buys over rounding it, and it
  is why the capacities are read back rather than reasoned about.

### ⚠ A CORRECTION TO THIS RELEASE'S OWN HEADER: the seg path has SLACK, and the first draft said it had none

The `⛔` block this release added over `sd_stroke_seg` read: *"Both land under `SD_PATH_CAP_MIN` = 8,
so each array is opened at 8 and neither can grow here — a fifth lineto added below without moving
these arguments would double an array instead (128 B and a copy), which is why
`programs/path_cap_test.cyr` group N asserts the capacities AND the counts after the call."*
**It is false, and `cyrius distlib` copies `src/` headers verbatim into `dist/sadish.cyr`** — so a
⛔-voiced measurement that is wrong reaches every consumer vendoring the dist. Caught in review.

`sd_stroke_seg` asks for `(5, 4)` and `SD_PATH_CAP_MIN` = 8 floors BOTH, so the path is built with
**3 spare verb slots and 4 spare point slots**. MEASURED at those same arguments:

| extra linetos | verbs / points | vcap / pcap | bytes | `sd_alloc` calls |
|---:|---|---|---:|---:|
| 0 (the site today) | 5 / 4 | 8 / 8 | 240 | 7 |
| +1 | 6 / 5 | 8 / 8 | 256 | 8 |
| +3 | 8 / 7 | 8 / 8 | 288 | 10 |
| +4 | 9 / 8 | **16** / 8 | 432 | 12 |

⇒ The "128 B and a copy" figure is right for the doubling itself; its **trigger is four more
linetos, not one**. The verb array first doubles at the NINTH verb (`moveto` + 7 `lineto`s +
`close`), the point array at the ninth point one lineto later.
⇒ The sentence was also wrong about its own tests: #180/#181 (the capacities) do not move until the
fourth extra lineto, so they are **not** what catches an added call — #182/#183 (the counts, 5 and
4) are. The header now states both, and **group N8 (#236-#252) pins the whole boundary** rather than
leaving it to prose: #240/#241 are asserted against the real site's own measured 240 B / 7 calls,
and #252 says the step from "exactly full" to "one past" is one `SdPoint` plus one 128 B doubling.
⚠ `sd_stroke_disc`'s header was re-checked with it and is correct as written: that path is exactly
full in both arrays, so `k < 17` without moving the arguments really does double both (1,112 B).

### ⛔ The refusal contract 0.7.2 established holds, unchanged

`sd_path_new_cap` returns **0** exactly where `sd_path_new` did, and both sites' 0-checks
(`if (rp == 0)` / `if (dp == 0)`) still turn that into `SADISH_ERR_OOM` rather than a store through
address 0. Group N #219-#232 walks the constructor's three requests one at a time under a hook that
grants K and then refuses: K = 0 refuses the record (1 request made), K = 1 the VERB array (2),
K = 2 the POINT array (3), K = 3 the first `SdPoint` (4) — `SADISH_ERR_OOM` at every one, for the
rect and for the disc, and the whole round stroker likewise; the next healthy stroke is
byte-identical to a clean one (#233-#235).
⚠ **The K sweeps in `programs/stroke_oom_test.cyr` did NOT shift, and that is a finding rather than
an oversight.** They are keyed to each call's allocation COUNT, measured at run time by `so_count`,
and this change moved bytes only. All 153 of its checks pass untouched.

### ⛔ Rendering does not move

agnos `refagree`: **BYTE-IDENTICAL on all 200 paths**. All 29 suites green with every assertion but
one unedited; rekha (23 suites) and dhancha (18) pass against this `dist/`. Group N #216-#218 pins
three absolute ink totals measured on BOTH trees — the closed 8x8 rect at 16,092 coverage units, the
cubic at 12,582, the 54-glyph label at 21,932 — so a stroke that quietly lost a piece to a growth
failure would fail them where a self-comparison could not.

### The one assertion that moved, and the figures that were comments

`programs/stroke_style_test.cyr` #13, `round_cost` **34,432 → 3,232** — the figure this item named as
the one that would move, and the only check in any suite that did. Four MEASURED figures living in
comments moved with it and were re-measured rather than left: `programs/alloc_test.cyr`'s group B
arena high-water (893,760 B / 44,688 B a round → **269,760 / 13,488**) and its group C stroke note
(34,432 → **3,232**), `programs/stroke_oom_test.cyr`'s group A heap note (126,176 → **13,144 B**,
canvas included), and `src/raster.cyr`'s `_sd_scratch_init` header (34,432 → **3,232**).

### ⚠ Eight of group N's checks compared two literals; seven are now relations between measurements

Review found that #179, #192, #202, #206, #207, #211, #212 and #216 as first written compared two
constant expressions — `check(4 * 240 + 4 * 568, 3232)` and the like. No sadish state reached either
side, so **none of them could ever fail**: building the suite as it then was against 0.7.2's
`src/stroke.cyr` failed 13 checks and not one of these eight. They inflated the group's count
without gating anything.

Seven were rewritten to relate values the suite MEASURED rather than values it restates — the
single-call costs are now carried in `n_seg` / `n_segc` / `n_disc` / `n_discc` and the totals are
asserted against them:

| was | is |
|---|---|
| `check(176 + 4 * 16, 240)` | `check(g_sz1 + g_sz2 + g_sz3 + 4 * 16, g_bytes)` |
| `check(312 + 16 * 16, 568)` | `check(g_sz1 + g_sz2 + g_sz3 + 16 * 16, g_bytes)` |
| `check(816 - 568, 248)` | `check(g_bytes - n_disc, 248)` |
| `check(4 * 240 + 4 * 568, 3232)` | `check(4 * n_seg + 4 * n_disc, g_bytes)` |
| `check(4 * 7 + 4 * 19, 104)` | `check(4 * n_segc + 4 * n_discc, g_calls)` |
| `check(8 * 240 + 9 * 568 + 7 * 16, 7144)` | `check(8 * n_seg + 9 * n_disc + 7 * 16, g_bytes)` |
| `check(8 * 7 + 9 * 19 + 7, 234)` | `check(8 * n_segc + 9 * n_discc + 7, g_calls)` |

⇒ These now gate something the byte totals alone do not: that the measured total **decomposes** into
the measured pieces, so a dropped piece, an extra piece, or a growth request hiding outside the
first three sizes fails them.
⚠ **They are corroboration, not adoption gates, and the re-measurement says so plainly.** Rebuilding
the rewritten suite against 0.7.2's `src/stroke.cyr` fails 15 checks — #174, #177, #178, #180, #181,
#187, #190, #191, #193, #194, #202, #204, #209, #214, #240 — and the six decomposition rows are NOT
among them, because a 4,208 B rect decomposes as honestly as a 240 B one (48 + 2,048 + 2,048 + 64 =
4,208). What they catch is a fourth path allocation appearing: MEASURED under the `(16, 16)`
capacity mutation, #192 reads 560 against a measured 816 while #189 (48 B) and #191 (128 B) still
read their exact expected sizes. Only #202 of the eight became an adoption gate outright.
⇒ The eighth, `check(16357904 - 1557176, 14800728)`, was **deleted**: its 0.7.2 side is history, not
a value this tree can measure, so no honest form of it exists. The arithmetic is kept as a comment
beside #214, which is the gate. Group N is 82 checks (#171-#252), and the suite 252.

### MUTATIONS — 17 single edits, 14 killed, 3 equivalent

Each applied alone, all 29 suites built and run against it, then the file restored and its MD5
checked against the pristine copy. The last four target `src/path.cyr` — the floor and the doubling
that group N8 measures — rather than the two changed call sites.

| mutation | killed by |
|---|---|
| `(5, 4)` → `sd_path_new()` | `path_cap_test` #174 (4,208), `stroke_style_test` #13 |
| `(17, 16)` → `sd_path_new()` | `path_cap_test` #187 (4,400), `stroke_style_test` #13 |
| `(5, 4)` → `(9, 4)` | `path_cap_test` #174 (248), `stroke_style_test` #13 (3,264) |
| `(5, 4)` → `(5, 9)` | `path_cap_test` #174 (248), `stroke_style_test` #13 (3,264) |
| `(17, 16)` → `(16, 16)` | `path_cap_test` #187 (816), `stroke_oom_test` #14 (47), `stroke_style_test` #13 |
| `(17, 16)` → `(17, 15)` | `path_cap_test` #187 (800), `stroke_oom_test` #14 (47), `stroke_style_test` #13 |
| `(17, 16)` → `(18, 16)` | `path_cap_test` #187 (576), `stroke_style_test` #13 |
| `(17, 16)` → `(16, 17)` | `path_cap_test` #187 (824), `stroke_oom_test` #14 (47), `stroke_style_test` #13 |
| `while (k < 16)` → `k < 17` | `path_cap_test` #187 (1,112), `stroke_oom_test` #14 (51), `stroke_style_test` #13, `flatten_bound_test` #360 |
| `if (rp == 0) return OOM` → `return OK` | `path_cap_test` #219, `stroke_oom_test` #31 |
| `if (dp == 0) return OOM` → `return OK` | `path_cap_test` #227, `stroke_oom_test` #16 |
| `SD_PATH_CAP_MIN` 8 → 9 | `path_cap_test` 37 checks incl. **N8 #245 #248 #250 #251 #252**, `flatten_bound_test` #120, `stroke_style_test` #13 (3,296) |
| `SD_PATH_CAP_MIN` 8 → 7 | `path_cap_test` 26 checks incl. **N8 #244 #245 #246 #249 #250 #251 #252**, `flatten_bound_test` #120, `stroke_style_test` #13 (3,168) |
| `sd_path_grow` `cap * 2` → `cap * 2 + 8` | `path_cap_test` 27 checks incl. **N8 #248 #250 #252**, `flatten_bound_test` #161 |

⚠ **`while (k < 16)` has TWO occurrences in `src/stroke.cyr`** — `sd_stroke_disc`'s vertex loop and
the styled stroker's 16-entry trig table — and a one-line anchor hits both. The row above is the
disc loop alone, anchored on the preceding `var vrc = SADISH_OK;`. A mutation harness that edits by
substring has to count its matches before it writes; this one asserts a single match and skipped the
edit rather than mutating two sites at once and reporting the result as one.

⚠ **Three mutations SURVIVED, and all three are genuinely EQUIVALENT:** `(5, 4)` → `(4, 4)`,
`(5, 4)` → `(5, 3)` and `(5, 4)` → `(6, 4)`. `SD_PATH_CAP_MIN` = 8 floors both of `sd_stroke_seg`'s arguments, so every
argument pair with each value ≤ 8 builds a bit-for-bit identical path — there is nothing for a test
to observe. No suite can kill them and none was written to pretend otherwise. What *is* asserted is
that the arguments are the site's TRUE counts: group N #182-#183 reads 5 verbs and 4 points back off
the path, so the pair stops being a free-floating number even where the allocator cannot tell it
from another. Above the floor the arguments bite in both positions — `(9, 4)` and `(5, 9)` are both
killed.
⚠ **The capacity read-backs (#180-#185, #193-#196) killed no mutation on their own.** Every capacity
error I could construct also moves a byte total or an allocation count, which #174/#175/#187/#188
catch first. They are kept because they state the ask directly — "the capacities after construction,
no growth" — and because they are what survives if a byte figure is ever re-based; they are reported
here as corroboration, not as unique gates.
⚠ **Group N8 is not a unique killer either, and one of its checks is.** Its capacity and byte rows
fire under all three `src/path.cyr` mutations above, but never alone — `#199`/`#201`/`#202` or the
group-L capacity checks catch those too. The exception is **#240**, which asserts the `+1 lineto`
cost against the REAL site's measured `n_seg`: rebuilding the suite against 0.7.2's
`src/stroke.cyr` fails 15 checks and #240 is one of them, so N8 is tied to the adoption and not
only to the constructor. The rest of N8 exists to keep the corrected header's numbers MEASURED
rather than asserted in prose — a comment cannot be mutation-tested, so the boundary it describes
is written down as checks instead.

### Not done (deliberately) — ⭐ DONE IN 0.9.0, and here is what those three figures became

0.8.0 wrote: *"Inline `(x, y)` storage … is still about half of what these two sites cost, as it was
of the glyph set (51 %): of `sd_stroke_disc`'s 568 B, **256 B (45 %) are the sixteen `SdPoint`
objects**; of `sd_stroke_seg`'s 240 B, 64 B (27 %); of the closed rect stroke's 3,232 B, **1,280 B
(40 %)**."* MEASURED after the change: the disc is **440 B in 3 calls** (was 568 in 19), the seg
**240 B in 3** (was 240 in 7 — the same bytes, re-spent from four records into eight inline slots),
and the closed 8x8 rect stroke **2,720 B in 24 calls** (was 3,232 in 104). The 54-glyph label is
**1,318,728 B in 12,806 calls**, from 1,557,176 in 50,593.

---

## Still open (items 1 and 2 closed; re-audited 2026-09-16 against 0.7.2, HEAD `cbb7229`)

1. ~~**"allocate the verb and point arrays at exactly the given capacity"**~~ — 🟢 **CLOSED in
   0.7.2**, see the section above: two capacity fields in the same 48 B record, 78,656 B on this
   proposal's own set. ⚠ The ask's companion phrase "same `SdPath` layout" survives in the sense
   that mattered — the record is the same size and every published offset below +40 is where it was.
2. ~~**No in-tree adopter.**~~ — 🟢 **CLOSED in 0.8.0**, see the section above. `sd_stroke_seg`
   (in `src/stroke.cyr`) calls `sd_path_new_cap(5, 4)` and `sd_stroke_disc` `sd_path_new_cap(17, 16)`
   — cited by function, not by line, as `issues/README.md` asks: an earlier draft of this sentence
   said `:208` for the second, which was the line of a comment, and both numbers moved again when
   the headers were corrected. A sweep of `src/` found no third known-size site (`src/dash.cyr`
   builds no `SdPath`). The item's 3,264 B estimate re-derives as **3,232 B in the SAME 104
   allocations** — it asked to be re-derived rather than quoted, and the 32 B is the four 8 B verb
   slots 0.7.2's split capacities took off the disc path. Gated by `programs/path_cap_test.cyr`
   group N (82 checks, #171-#252) and `programs/stroke_style_test.cyr` #13.
   ⚠ The cross-reference to
   `issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md` is fully
   settled now: that filing's half (both results 0-checked) closed in 0.7.2, and this half (the
   capacity) closes here. Both sites' `if (rp == 0)` / `if (dp == 0)` guards are unchanged and still
   gated — `sd_path_new_cap` refuses with the same 0 `sd_path_new` did.
3. **rekha adoption** — *"rekha updates that test when it adopts the new call"* is unverifiable from
   this repo, and nothing here tracks it. ⛔ **This is why the filing does not archive**, and 0.8.0
   does not change it: rekha's `rekha_outline_to_sdpath` still calls `sd_path_new`, and neither fact
   can be asserted from here — rekha's 23 suites were run against this `dist/` and pass, which says
   the change is compatible, not that the ask is met. ⚠ sadish's own adoption (item 2) is evidence
   the call is usable, not a substitute for this.
   ⚠ **And the "test" the ask names is looser than this filing has been calling it.** Re-read on the
   live tree 2026-09-16: `rekha/programs/alloc_test.cyr` carries 4,192 B as a MEASURED figure in the
   comment beside its check #5 (*"4,192 B of sadish path (`sd_path_new` 4,144 + 3 points x 16)"*);
   the assertion itself is `check(glyph_cost > 0, 1)`. So adoption there would move a documented
   number and a `> 0` bound, not a literal pin — earlier drafts of this section said "pins", which
   overstates what rekha would have to edit. The ask still stands; only its size was wrong here.
