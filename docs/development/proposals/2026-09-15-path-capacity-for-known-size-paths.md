# Capacity-sized paths: `sd_path_new_cap(n_verbs, n_points)` for callers that know the path's size

**Status:** 🟡 **STILL OPEN — the CAPABILITY is complete as of 0.7.2, the ADOPTION asks are not.**
0.7.2 gives the verb and point arrays separate capacities (`SD_PATH_CAP_OFFSET` = verbs,
`SD_PATH_PCAP_OFFSET` = points, in the 48 B record's former `reserved` word), so
`sd_path_new_cap(n_verbs, n_points)` now opens each array at exactly its own requested capacity and
each doubles alone: **MEASURED 78,656 B on this proposal's own ASCII set, its target to the byte**
(`programs/path_cap_test.cyr` #160, 170 checks, 24 single mutations each killed). That closes item 1
of **Still open** and nothing else — items 2 (no in-tree adopter: `sd_stroke_seg` / `sd_stroke_disc`
still call `sd_path_new`) and 3 (rekha adoption) are untouched, so ⛔ **this filing does not archive
yet.** Was 🟡 OPEN — SHIPPED in 0.7.1 with a deviation from this proposal's first bullet, and 🟡 OPEN
— a capability request before that. It read "🟢 **CLOSED — SHIPPED**" until the 2026-09-15 re-audit
against HEAD `f722e8c`.
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

### Not done (still open, deliberately)

Inline `(x, y)` storage instead of `SdPoint` pointers — the "worth measuring alongside" section
above. It is an ABI change and was out of scope for 0.7.1. Its number stands: 39,968 of the 78,656
exact-capacity bytes are `SdPoint` objects. Re-verified 2026-09-15: `SdPath` still stores pointers
(the `SdPath` layout comment, `src/path.cyr:50` at 0.7.1, `:51` at 0.7.2) and `sd_path_push_point`
still takes one (`src/path.cyr:172` at 0.7.1, `:221` at 0.7.2); check #132 pins the 39,968. This
proposal labels it "not requested here", so it is declared residue rather than a broken promise —
but on this file's own numbers it is the largest item left (58,672 B vs 78,656 B).

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

## Still open (item 1 closed 0.7.2; re-audited 2026-09-15 against 0.7.1, HEAD `f722e8c`)

1. ~~**"allocate the verb and point arrays at exactly the given capacity"**~~ — 🟢 **CLOSED in
   0.7.2**, see the section above: two capacity fields in the same 48 B record, 78,656 B on this
   proposal's own set. ⚠ The ask's companion phrase "same `SdPath` layout" survives in the sense
   that mattered — the record is the same size and every published offset below +40 is where it was.
2. **No in-tree adopter.** ⚠ Re-checked 2026-09-15 after the item-1 change: the callers of
   `sd_path_new_cap` are `programs/flatten_bound_test.cyr` and now `programs/path_cap_test.cyr` —
   still both TESTS, so this item stands in full. ⚠ Its 3,264 B figure was computed under ONE
   capacity and is now 8 B per disc path lower (`sd_path_new_cap(17, 16)` opens 48 + 136 + 128 =
   **312 B**, not 320; `(5, 4)` is 176 B either way — MEASURED, `programs/path_cap_test.cyr` #30-31);
   re-derive it when the adoption is made rather than quoting it. sadish's own two known-size path
   sites still call `sd_path_new`:
   `sd_stroke_seg` (5 verbs / 4 points, `src/stroke.cyr:171` at 0.7.2) and `sd_stroke_disc`
   (17 verbs / 16 points, `:185`). MEASURED: one round stroke of a 4-vertex rect costs
   **34,432 B in 104 allocations** today (pinned at `programs/stroke_style_test.cyr:694`); with those
   two sites on `sd_path_new_cap(5, 4)` and `sd_path_new_cap(17, 16)` it is **3,264 B in the SAME 104
   allocations** — 10.5× less, every other suite still green, only that pinned figure moving. Not
   something this proposal asks for, but it is exactly the arena cost it was opened about.
   ⚠ Those same two sites are the ones
   `issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md` filed as unchecked
   `sd_path_new()` results; one edit closes both. ⚠ **Half of that cross-reference is stale at 0.7.2,
   checked 2026-09-16:** both results ARE checked now (`if (rp == 0) { return SADISH_ERR_OOM; }` /
   `if (dp == 0) { … }`, `src/stroke.cyr:172` / `:186`) and that filing is closed. What stands here is
   the CAPACITY ask alone, which is untouched — both sites still call `sd_path_new`, not
   `sd_path_new_cap`.
3. **rekha adoption** — *"rekha updates that test when it adopts the new call"* is unverifiable from
   this repo, and nothing here tracks it.
