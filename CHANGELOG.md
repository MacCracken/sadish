# Changelog

All notable changes to sadish are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.11.0] - 2026-09-20 — pattern paint, an API reference, and the device line gets a gate

Three roadmap items close: the **image/pattern paint** capability gap, the **"no API reference"**
infrastructure gap, and the `sd_present_open` device line that 0.8.0 wrongly wrote off as
untestable. No ABI break — `SdGradient` grows a fourth *kind*, not a field. 34 RUN suites green
(32 + two new); `lint`, `vet`, `fmt`, `distlib --check` and the aarch64 + AGNOS cross-builds clean.

### Added — pattern paint: a paint that samples a surface

```
sd_pattern_new(src)                  sd_paint_is_pattern(p)     sd_pattern_source(p)
sd_pattern_set_filter(p, f)          sd_pattern_filter(p)
```

A **pattern** is a fourth paint kind (`SD_PAINT_PATTERN`) beside the three gradients. It shares the
record, the spread, the matrix and both blit entries — `sd_canvas_blit_paint_at` and
`sd_canvas_blit_paint_premul_at` take it unchanged — and differs only in where a pixel's colour
comes from: one **source texel**, nearest-neighbour, instead of a stop ramp. A consumer can now draw
a bitmap, a repeating tile or a cached atlas, which is what a canvas/SVG layer could not do at all.

⭐ **A pattern is ONE allocation**: the 88 B record. It has no stops and no ramp, so it skips the
4,164 B a gradient spends on those — and the blit dispatches on the kind BEFORE reading either
field, which is what makes that safe. ⭐ **A pattern blit allocates nothing**, the 0.10.0 property,
held for the new kind on both the plain and matrix-mapped paths.

⚠ **The source is borrowed, not copied.** It must outlive the pattern, and mutating it changes what
the pattern paints. Under a per-frame arena, a pattern must not outlive the surface it samples.

⚠ **Nearest only, but the filter is a FIELD, not an assumption.** `sd_pattern_set_filter` refuses
anything else with `SADISH_ERR_UNSUPPORTED` rather than silently doing nearest, so bilinear can land
later without an ABI break and a consumer that asks for a filter it did not get is told. Nearest is
the right default and not merely the cheap one: at 1:1 — a cached tile, an icon, a glyph atlas — it
is exact, and smoothing there is a defect.

### Added — `SD_SPREAD_NONE`, and a refusal that finally means something

A fourth spread: **paint nothing outside the source rect** (SVG's `pattern`, Canvas's `no-repeat`).
Without it a consumer could tile or smear edge pixels but never draw one bitmap once.

⛔ **It SKIPS the pixel rather than writing it transparent**, and the difference is real: the
straight blit forces destination alpha 255, so a "transparent write" still stamps an opaque pixel
and loses what the destination had. Gated directly (`programs/pattern_test.cyr` group E).

⛔ **A gradient cannot be set to it** — `sd_gradient_set_spread` answers `SADISH_ERR_UNSUPPORTED`,
and that is the **first use of that code anywhere in the library** (it had zero return sites and
existed only in the enum). A gradient's `t` is defined everywhere on the plane, so there is no rect
to be outside of; padding it silently would paint the last stop over the whole canvas. ⚠ A value
merely outside the enum is still `SADISH_ERR_BOUNDS` — the two refusals differ on purpose.
⚠ **Behaviour change:** `sd_gradient_set_spread(g, 3)` returned `SADISH_ERR_BOUNDS` through 0.10.0,
because 3 was one past the last spread. It is `SADISH_ERR_UNSUPPORTED` now.

### Fixed — a signed-shift bug the new suite caught before it shipped

⛔ The pattern sampler's first draft reduced a 16.16 pattern-space coordinate to a texel index with
`>> SD_SHIFT`. **Cyrius's `>>` is LOGICAL**, so every canvas pixel left of or above a translated
pattern — exactly what a translating matrix produces — had its sign bits zero-filled and became an
enormous positive index. MEASURED as that mutation: the three pixels left of the origin read texels
1, 2, 0 under `REPEAT` instead of 0, 1, 2, and 2, 2, 2 under `PAD` instead of 0, 0, 0. The fix is
`sd_asr`; `programs/pattern_test.cyr` group D is the gate, and it exists because a negative texel
index is only reachable through a matrix, which no other suite builds.

### Fixed — `sadish_version()` was stale for two releases

⛔ It returned **900** through 0.9.1 AND 0.10.0 — including across the 0.10.0 **ABI break**, which is
precisely the release a version probe exists to let a consumer detect. `programs/integration_test.cyr`
asserted the same stale literal, so the gate agreed with the bug and no suite could see it.
⇒ It returns **1100**, and CI now **derives the expected number from the `VERSION` file** and fails
if either the function or the test disagrees. The number cannot drift again.

### Added — `docs/api.md`, a real API reference

VERIFIED at 0.10.0: every contract lived in a source header, so a consumer learned the library by
reading `dist/sadish.cyr` — now 8,000+ lines. `docs/api.md` covers all **122** public functions by
module, and front-loads the six cross-cutting rules most consumer bugs come from: 16.16 coordinates
and why `>>` is not the shift you want; the allocation seam and what a refusal costs; **what a hook
can still refuse after 0.10.0**; the integer error model and its overloaded codes; the
0-means-opaque alpha rule and the two alpha worlds; and that none of it is thread safe.

⚠ `cyrius doc` was not used: it captures only the LAST LINE of each doc comment, which for this
tree's multi-paragraph headers is usually a fragment. The reference is written, and the source
headers stay normative — where the two disagree, the header is right and the reference is a bug.
⭐ Its worked example is compiled and run, and every constant in its table is machine-checked.

### Added — `programs/present_geom_test.cyr`: the device line, gated at last

`sd_present_open` — the two tokens `"/dev/fb0"` — was reached by no test in the tree.
`present_open_test` drives everything BELOW the device name over a regular file and says so in
capitals; 0.8.0 recorded the rest as "needing a display" and wrote it off. ⛔ **That was wrong:**
`sd_present_open` opens, probes and allocates but paints NOTHING. Only `sd_present_blit` writes
pixels, and only that call ever needed the "no test may touch the live display" rule.

The new suite opens the real framebuffer, asserts the presenter's geometry against **its own
independent ioctl** on its own read-only descriptor (a shared helper would agree with sadish by
construction and prove nothing), checks the descriptor is held and then given back, opens twice, and
closes. VERIFIED on a 2560x1440 32bpp device: pitch 10240, nothing drawn.
⚠ It **SKIPS, loudly**, where there is no framebuffer or no permission — CI has no device and
`/dev/fb0` is `root:video` — printing which branch it took, because a skip that reads like a pass is
worse than no gate. Both branches verified. ⛔ It never calls `sd_present_blit`, and that line is
the only thing between this suite and a test that draws on the user's screen.

## [0.10.0] - 2026-09-20 — SdPolyline stores its points inline, and a path can transform and bound itself

⚠ **ABI BREAK**, the one `docs/development/roadmap.md` filed under "What 1.0 freezes". 0.9.0 put
`SdPath`'s points inline and deliberately left `SdPolyline` alone — *"a second ABI break in one
release is how a consumer stops trusting the version number"* — and that deferral had a visible
price: because the flatten OUTPUT still held pointers, what the path stopped paying per point the
flatten started paying. This closes it. Two additive functions ride along; both are pure additions
and break nothing.

⛔ **Rendering does not move.** A 255-measurement oracle across fills (both rules), round / styled /
dashed strokes, clips and flatten geometry on **both AA engines** diffs to **zero** against 0.9.1 on
every ink total, weighted ink checksum, point count, verdict and flattened-coordinate checksum —
**225 of 255 measurements byte-identical, and the 30 that moved are all `flat_bytes` / `flat_calls`,
which is the release.** 32 RUN suites green (31 + the new one); `lint`, `vet`, `fmt`,
`distlib --check` and the aarch64 + AGNOS cross-builds clean.

### Changed — a flattened point costs no allocation

`SdPolyline`'s points array holds the coordinates **INLINE — x at +0, y at +8, 16 B a slot** — where
it held 8 B pointers to separately allocated 16 B `SdPoint`s. The 24 B header and its three offsets
do not move; only the array's stride does.

⭐ **The per-point allocation leaves the library entirely.** MEASURED, each from its own suite:

| | 0.9.1 | 0.10.0 |
|---|---:|---:|
| styled stroke of a cubic, on the seam | 128 B | **0** |
| dashed stroke of a cubic | 224 B | **0** |
| fill of the hostile 4,096-quad path | 1,110,016 B / 69,376 calls | **0 / 0** |
| `sd_path_flatten`, 20,001 points | 20,004 calls | **5** |
| 4,096 curves under a 64-point budget | 4,165 calls | **4** |
| 54-glyph label, round-stroked | 1,318,728 B / 12,806 calls | 1,296,040 B / 11,388 |

⇒ Nothing in a flatten is per-point or per-curve any more; what is left is **per-doubling** of the
output array. A fill of any path, and a styled or dashed stroke of any path, now draw through a
consumer's `sd_alloc` hook **without asking it for a single byte**.

⚠ **`SD_FLATTEN_PCAP` = 4096 is new** — the initial capacity of the three point stores
(`sd_path_flatten`'s output and the fill's and stroker's per-curve buffers), halved as the slot
doubled, exactly as `SD_PATH_PCAP` was for 0.9.0. At 4096 × 16 the first allocation is **65,536 B,
byte-identical to every release since 0.7.0**. `SD_FLATTEN_CAP` = 8192 still sizes the stores whose
slot did not change: the fill's edge list and crossings, and the stroker's run.

⚠ **What the wider slot also does, said plainly.** A slot is 16 B where it was 8, and reaching a
given capacity takes one more doubling — so on a bump arena with no `free()`, where every superseded
array stays charged, the CUMULATIVE bytes of a LARGE flatten go **up**: a 20,001-point flatten is
778,816 B → 983,088. The two figures describing memory a consumer actually holds both improve —
**live bytes 582,160 → 524,288** and **calls 20,004 → 5** — and a consumer whose arena is reset per
frame pays the live figure. The trade was taken deliberately: halving `SD_FLATTEN_PCAP` keeps the
common case (a glyph, a small path) on the same 65,536 B first block rather than 131,072.
`programs/grow_edges_test.cyr` C7 works it through in full.

### Added — the accessors that make the break survivable

```
sd_polyline_point_x(pl, i)    sd_polyline_point_y(pl, i)
```

⚠ No bounds check — bound the loop on `sd_polyline_count`. ⭐ These exist for exactly the reason
`sd_path_point_x` / `_y` did at 0.9.0: through 0.9.1 there was **no accessor**, so every reader in
this tree and every consumer open-coded `sd_point_x(load64(sd_polyline_points(pl) + i * 8))`. That
spelling now reads an x COORDINATE as an address — **an immediate SIGSEGV** for any path off the
origin, and a silently wrong answer for one on it. `sd_polyline_points` stays published for a caller
that walks the block itself, which must now stride `SD_POLYLINE_PT_SIZE` (= 16).

### Added — `sd_path_transform` and `sd_path_bounds`

Both VERIFIED missing at 0.9.1 and hand-rolled by every consumer; both pure additions.

```
sd_path_transform(path, m)     # affine every point IN PLACE; SADISH_OK / SADISH_ERR_BOUNDS
sd_path_bounds(path, out)      # 4 x 16.16 into a 32 B buffer; SADISH_ERR_EMPTY_PATH if no points
```

⭐ **`sd_path_transform` allocates NOTHING**, which is the whole reason it exists: `sd_matrix_apply`
takes and returns an `SdPoint`, so the loop a consumer writes by hand costs one 16 B record per point
plus a second path to hold the result. MEASURED on a 7-point path: the hand-rolled shape makes **14
`sd_alloc` calls / 224 B**, this makes **0 / 0**. It is the same arithmetic as `sd_matrix_apply` to
the raw unit — gated point-by-point against it across five matrices.
⚠ IN PLACE: the original geometry is gone. It transforms the CONTROL points, which is exact for
affine (the affine image of a Bézier is the Bézier of the images of its control points), so curves
are not re-approximated.

⚠ **`sd_path_bounds` is the control-point hull, not the ink.** A Bézier lies inside its control
polygon and need not touch it, so a path with curves reports a box that CONTAINS the drawn shape and
may exceed it — the conservative answer culling, damage tracking and layout all want. Gated as a
real superset: every point of the flattened path falls inside the reported box, and the box is
strictly larger for a curve that bows away from its control. ⛔ Recomputed per call, not cached:
`SdPath`'s 48 B record has no spare word (0.7.2 spent the one annotated *"bounds-cached"* on the
point capacity), so caching would be an ABI break of its own.
⚠ An empty path returns `SADISH_ERR_EMPTY_PATH` and **`out` is not written** — a zeroed box would
read as a real degenerate box at the origin, and a culler would keep the object instead of dropping
it.

### Changed — what a hook can still refuse

⛔ **A fill and a styled or dashed stroke can no longer be starved by a consumer's hook.** They still
return `SADISH_ERR_OOM` where the code says they do, but the reachable paths are now the process
scratch and the edge list, which draw from the GLOBAL allocator by design and were never a hook's to
refuse. A consumer testing its own error path by refusing from its hook will get a complete picture
and `SADISH_OK`. ⚠ The ROUND stroker is the exception — its per-segment and per-vertex piece paths
are real `sd_alloc` calls and starve exactly as before. `programs/stroke_oom_test.cyr` holds both
halves, and `src/alloc.cyr`'s seam contract says so.

### Added — `programs/transform_test.cyr` (71 checks)

The gate for all three additions, and for the one thing no other suite covers: that the polyline
accessors agree with a RAW strided read of the block, which is the read a consumer walking the array
itself will write. Also holds `(0, 0)` as a real point in the flatten output — the same trap 0.9.0's
inline points sprang on paths, where a null test used to mean "no point".

### Fixed — a test that was passing for two reasons at once

`programs/stroke_oom_test.cyr` #108 compared a ROUND/MITER stroke against reference case **4**
(BUTT/BEVEL) and asserted only that the two DIFFERED — which they did whether or not the starvation
under test worked. Now that the stroke cannot be starved, the comparison names the right picture
(case 5) and asserts equality.

## [0.9.1] - 2026-09-20 — the 6.6.6 pin, and the two things the gates were not catching

A patch release: **no API, no ABI, no rendering change.** The toolchain pin moves 6.6.4 → 6.6.6 and
`lib/` is re-resolved against it. The pin bump itself needed no source change — the roadmap's
pre-flight (zero `struct` declarations, no `async`/`operator` fns, no `ret2`/`rethi` pairs, no
top-level `{ }` blocks, no locally defined `vec_*`) held item for item. What the bump *did* do is
surface two defects the CI gates had been passing over, and both are fixed here. All 31 RUN suites
green; `lint`, `vet` and `distlib --check` clean.

### Fixed — `cyrius fmt --check` was a false negative, so the format gate was blind

VERIFIED: `programs/paint_focal_test.cyr` had three continuation lines indented 12 spaces where the
6.6.6 formatter wants 10 — and `cyrius fmt <file> --check` **exited 0 on it anyway**, while
`cyrius audit`'s fmt stage failed the same file. The drift is genuinely 6.6.6's: MEASURED, the 6.6.4
formatter rewrote the file to zero lines, the 6.6.6 formatter to three.

⛔ **The file is the small half of this.** CI's "Format check" step gated on that `--check` exit code,
so the gate could not have caught the drift it was there to catch — any 6.6.6 re-indent would have
landed on `main` unreported. The step now formats in place on the clean checkout and lets
`git diff --exit-code` report, which cannot silently agree: `lib/` is gitignored, so only tracked
source can dirty the tree at that point. ⚠ The `--check` flag is left unused rather than trusted.
VERIFIED after the fix by formatting a COPY of every file in `src/` and `programs/` and diffing it
against the original: **0 of 46 files drift.**

### Fixed — raw x86_64 syscall numbers in `src/`, byte-for-byte

VERIFIED: the aarch64 build emitted exactly one warning, and it was a real portability bug —
`src/present.cyr:376` issued `syscall(8, …)` for `lseek`, and **8 on ELF-aarch64 is `getxattr`**. The
roadmap had already predicted this class ("the hard-coded `2`/`1` are also wrong on aarch64"); 6.6.6's
sharpened diagnostic names the colliding call, which is what made it worth closing now. ⚠ Pre-existing,
not introduced by the bump — 6.6.4 warned too, less precisely.

**11** sites in `src/` (10 in `present.cyr`, 1 in `error.cyr`) now take the stdlib's target-dispatched
constants — `SYS_WRITE` / `SYS_CLOSE` / `SYS_LSEEK` — instead of `1`/`3`/`8`.
⭐ **MEASURED that this moves nothing on the shipped target:** builds are byte-deterministic (verified by
building one program twice), and **all 32 binaries — every `programs/*.cyr`, smoke included — are
sha256-identical before and after the substitution.** On x86_64 the constants *are* 1/3/8, and the
codegen proves it rather than asserting it.

⛔ **The sweep stops at three of five, and the reason is the interesting part.** `open` and `ioctl` have
NO spelling that exists on every target sadish builds for: **aarch64 has no `open` at all** (only
`openat`, so the stdlib defines no `SYS_OPEN` for it) and **AGNOS's ABI has no `ioctl`** (no
`SYS_IOCTL`). MEASURED, not assumed: naming them fails the `--aarch64` and `--agnos` link-checks
outright — `undefined variable 'SYS_OPEN'` and `undefined variable 'SYS_IOCTL'`. That is strictly worse
than a wrong number on a path that cannot run on those targets anyway, because everything reached
through those four calls is the **Linux `/dev/fb0` sink**, which this file's header has always scoped as
Linux-only; the AGNOS sink is a separate backend (kernel `blit#39`), not a port of this one. They stay
raw, with the reasoning written at the call site so the sweep is not "finished" by a later reader.

⚠ **This is the first real use of `lib/syscalls.cyr` from `src/`** — at 0.9.0 nothing in `src/` referenced
a `SYS_*` or `sys_*` symbol. It adds no consumer burden: `syscalls` was already in `[deps] stdlib` and
already listed in the `dist/sadish.deps` sidecar. VERIFIED by compiling a consumer-style program that
includes the nine declared stdlib leaves plus `dist/sadish.cyr` and calls through both changed paths
(`sd_surface_write_ppm` → `SYS_WRITE`/`SYS_CLOSE`, `sadish_err_print_name` → `SYS_WRITE`): links and
runs clean.

⚠ **`programs/` keeps its raw numbers** (`syscall(2, …)` in `present_open_test.cyr`, `oom_test.cyr`,
`stride_test.cyr` and others). Deliberate: the suites are host-only and never cross-compiled, so the
aarch64 collision cannot reach them. Only the shipped library is portable.

### Added — CI cross-builds aarch64 and AGNOS, and fails on the warning that started this

The portability fix above rots unless something gates it, and the roadmap's standing
"CI builds the host target only" gap is exactly why nothing would have. A new step link-checks
`programs/smoke.cyr` for **`--aarch64` and `--agnos`** and **fails on the compiler's own `raw syscall`
diagnostic**, so a re-introduced raw number is caught at the PR rather than at a port.
⚠ Link-check only — CI has neither an aarch64 nor an AGNOS host, so the binaries are built and not run;
the RUN suites stay x86_64. VERIFIED locally: all four targets (`x86_64`, `--aarch64`, `--agnos`,
`--win`) build with **zero** raw-syscall warnings. ⭐ AGNOS is the point of the stack and this is the
first time anything has proven sadish still compiles for it.

### Changed — toolchain

- `cyrius` pin **6.6.4 → 6.6.6** (`cyrius.cyml`).
- `lib/` re-resolved from the 6.6.6 store. ⭐ This also clears the standing
  `warning: ./lib/ shadows version-pinned …/lib — 12 bundled lib(s) differ` that every build carried
  at 0.9.0: the vendored fold had drifted 12 libs behind. The build is now diagnostic-free on the host.

## [0.9.0] - 2026-09-16 — SdPath stores its points inline

One change, shipped alone because it is an ABI break: `SdPath`'s points array holds the coordinates
INLINE — x at +0, y at +8, **16 B a slot**, at the same `SD_PATH_POINTS_OFFSET` in the same 48 B record
— instead of 8 B pointers to separately allocated 16 B `SdPoint`s. rekha filed the measurement; this is
it implemented.

### Changed — a point costs 16 B and no allocation

⭐ **The proposal's target, hit to the byte.** 95 ASCII-shaped paths (1,768 verbs, 2,498 points) built
through the public builders at exact capacity, on the allocation seam:

| | bytes | `sd_alloc` calls |
|---|---:|---:|
| 0.8.0 | 78,656 | 2,783 |
| 0.9.0 | **58,672** | **285** |

A path is **three allocations** — record, verb array, point array — whatever it holds; 0.8.0 made three
plus one per point. Downstream, MEASURED: the closed 8x8 rect stroke **3,232 B → 2,720 B** in 104 → 24
calls, and the 54-glyph round-stroked label **1,557,176 B → 1,318,728 B** in 50,593 → 12,806.
⛔ **Rendering does not move.** agnos's `refagree` prints **BYTE-IDENTICAL on all 200 paths**, and two ink
oracles (365 and 416 measurements across fills, both strokers, dashes, clips and paint on both AA
engines) diff to zero against 0.8.0.

### Added — the accessors that end the coupling

```
sd_path_point_x(path, i)    sd_path_point_y(path, i)    sd_path_verb_at(path, i)
```

⚠ No bounds check — bound the loop on `sd_path_point_count`. ⭐ These exist because the ABI break was
only reachable at all through open-coded offsets in consumer code: rekha's 14 reads exist because there
was no accessor. `sd_path_verb_at` has no internal caller and ships anyway, so a consumer walking a path
carries no offsets in either stream.

### Changed — constants that moved with the slot

- `SD_PATH_PCAP` = 128 is new: `sd_path_new`'s default POINT capacity, halved as the slot doubled, so
  `sd_path_new`'s first allocation is the same **4,144 B in 3 calls** it has been since 0.4.0.
  `SD_PATH_CAP` = 256 now sizes only the verb array.
- `SD_PATH_CAP_MAX` 2^28 → **2^27**, re-derived for the 16 B slot (2^27 × 16 = 2 GiB = `ALLOC_MAX`).
  ⚠ One constant bounds both arrays and takes the tighter slot, so a VERB capacity in (2^27, 2^28] is
  now refused where 0.8.0 served it.
- `SD_GROW_LIMIT_DEFAULT` 8 MiB → **16 MiB**. The ceiling is in BYTES and a run point went 8 B → 16 B,
  so at 8 MiB the stroker's run and the dash buffer would have held 524,288 points where 0.8.0 held
  1,048,576 — and a single subpath longer than that, which 0.8.0 stroked whole, would have truncated
  (MEASURED at a lowered ceiling: 0.8.0 `SADISH_OK` / 526,870 ink, an 8 MiB 0.9.0 `SADISH_ERR_OOM` /
  434,050). Doubling the default keeps the strokable length exactly what it was.
  ⭐ And it costs no real memory: MEASURED on both trees, a stroked point costs **40 B** of path + run
  on either release (0.8.0 pays 8 to the run and 32 to the path; 0.9.0 pays 16 and 24), so both
  truncate holding 1,048,576 × 40 B = 41,943,040 B.
  ⚠ **What the doubling also does**, said plainly: the ceiling bounds every growable store, so the
  fill's edge list can reach 524,288 edges where 0.8.0 stopped at 262,144 — a looser hostile-path bound,
  and the price of keeping stroke length stable across the ABI change. A consumer that wants 0.8.0's
  MEMORY bound rather than its length calls `sd_grow_limit_set(8388608)`. Pinned by `grow_edges_test`
  groups R and R2.

### Not in scope, deliberately

`SdPoint`, `geom.cyr` and `SdPolyline` are unchanged. The polyline still holds `SdPoint` pointers and
still carries the 0.8.0 verdict word. **A second ABI break in one release is how consumers stop trusting
a library**, and a curve's flattening still needs `SdPoint`s internally — which is why the one-quad
fill's seam cost went 48 B in 3 calls to 64 B in 4: the curve's own end point used to be a record the
path already owned.

### ⚠ Consumers — rekha must port, dhancha need not

- **dhancha: 18/18 pass.** It never reads `SdPath` internals.
- **rekha: 19 of 23 pass.** The four that fail — `cff_test`, `glyf_edge_test`, `hostile_test`,
  `path_start_test` — walk the points array and dereference `SdPoint`s; each is a SIGSEGV at the first
  read, not a silent wrong answer. `bench_hotpath` faults the same way. rekha's LIBRARY is unaffected:
  `src/` names no `SD_PATH_*` and calls no `sd_point_*`, and its smoke program builds and runs clean.
- ⭐ Filed for them with the port, site by site:
  `rekha/docs/development/issues/2026-09-16-sadish-0.9.0-inlines-path-points-five-programs-must-port.md`.
  The replacements are one accessor call each, and the index replaces the byte offset — `pts + 2 * 8`
  becomes `sd_path_point_x(pc, 2)`, so the `* 8` disappears rather than becoming `* 16`.

### Verified

All **31** suites pass, including the new `inline_points_test` (171 checks). `fmt --check` clean, `lint`
0 warnings, `vet` clean, `distlib` in sync with no duplicate top-level names. `sadish_version()` → **900**.
⭐ The reviews found ten issues, four of them ≥ medium — among them that a QUADTO fill-seam argument swap
was gated by `area_test` alone, and that `_sd_sb_restart`'s y coordinate had no check that would catch
`cx` written into it. Both are now killed by `inline_points_test` group L.

## [0.8.0] - 2026-09-16 — the repair backlog closes

The last three asks any filing still had. ⛔ **All four issue filings are now archived**, and
`docs/development/issues/` holds only its README; the one document still open is rekha's
`sd_path_new_cap` proposal, whose remaining item is an adoption this repo cannot verify. Rendering does
not move: agnos's `refagree` prints **BYTE-IDENTICAL on all 200 paths**, and rekha (23 suites) and
dhancha (18) pass against this `dist/`.

### Changed — the round stroker's piece paths are opened at their exact size

`sd_path_new_cap` shipped in 0.7.1 and had **no caller in `src/` at all** — only two test suites — while
`sd_stroke_seg` and `sd_stroke_disc` each bought `SD_PATH_CAP` = 256 verbs + 256 points (4,096 B of
arrays plus the record) for a path of five, or of seventeen, slots — per SEGMENT and per VERTEX of every
round stroke, on the consumer's seam. They now call `sd_path_new_cap(5, 4)` and `sd_path_new_cap(17, 16)`,
the counts derived from the calls that fill each path.
⭐ MEASURED on the seam, every row in the SAME number of `sd_alloc` calls:

| | 0.7.2 | 0.8.0 | | calls |
|---|---:|---:|---:|---:|
| one segment rect | 4,208 B | **240 B** | 17.5x | 7 → 7 |
| one 16-gon disc | 4,400 B | **568 B** | 7.75x | 19 → 19 |
| closed 8x8 rect stroke | 34,432 B | **3,232 B** | 10.65x | 104 → 104 |
| round stroke of a cubic | 73,376 B | **7,144 B** | 10.27x | 234 → 234 |
| 54-glyph label, round-stroked | 16,357,904 B | **1,557,176 B** | 10.50x | 50,593 → 50,593 |

⛔ **The allocation COUNT is the proof.** `sd_path_new_cap` makes the same three requests `sd_path_new`
did, for smaller blocks, so an unchanged count means neither array doubled and no piece was dropped.
`path_cap_test` group N reads the capacities and counts **off the record the hook handed the real site**:
the disc's path is 17 verbs of 17 and 16 points of 16, exactly full — one slot short, at (16, 16), costs
816 B in 20 allocations against 568 in 19.
⚠ The segment path is NOT tight: `SD_PATH_CAP_MIN` = 8 floors its (5, 4), so it carries 3 spare verb and
4 spare point slots, and the verb array first doubles at the NINTH verb. Group N8 pins that boundary,
because the first draft of the header stated it wrong and `cyrius distlib` ships `src/` headers verbatim
into every consumer's bundle.
**One assertion moved in 29 suites**: `stroke_style_test` #13, 34,432 → 3,232 B.

### Fixed — a flattened polyline says whether it is whole

The flatten filing's item 1 asked for "a flag on the polyline … so a truncated contour is refused or
clearly degraded". 0.7.1 and 0.7.2 shipped the parenthetical (the fill and all three stroke entries
return `SADISH_ERR_OOM`); the RESULT still said nothing, and the two witnesses that existed are sticky by
design, so a consumer holding a polyline taken a while ago could not ask about it.
`SdPolyline` is now **24 B**: points, count, and a verdict word at +16 carrying `SD_POLYLINE_TRUNCATED`
and `SD_POLYLINE_DEGRADED`, frozen when `sd_path_flatten`'s walk ends. New: `sd_polyline_truncated`,
`sd_polyline_degraded`, `sd_polyline_verdict`.
⚠ **The prefix is FLAGGED, not refused** — the bullet said "refused OR clearly degraded", and this is the
other branch, taken on three grounds: a starved flatten's prefix is byte-identical to the whole
flattening's first N points and a consumer recovering from an out-of-memory frame still wants it; `0`
already means two things here (empty path, refused allocation) and a third would deepen the confusion
this release repairs; and "refuse" belongs at the DRAW, where it has been since 0.7.1.
⚠ **`degraded` is the CALL's verdict where `sd_flatten_degraded()` is the OPERATION's.** Two flattens
sharing one `sd_flatten_op_begin` budget report separately: the one whose curve was cut says 1, the whole
one says 0, while the global says 1 for both.
⛔ **The call-scoped flag is cleared at the TOP of `sd_path_flatten`, not the bottom**, because the
recursions that set it are also reached by `sd_flatten_quad` / `_cubic` — the road a fill and the stroker
take. MEASURED with the clear displaced by a few lines: all 29 suites stayed green while a budget-cut
fill followed by a whole flatten returned a record stamped with the fill's verdict.

### Fixed — the presenter's own `open()` is reachable without a display

0.7.1 split `_sd_present_probe` and `_sd_presenter_build` out so a regular file's fd could drive their
allocation guards; the function they were split OUT OF stayed unreachable, because its first act is
`open("/dev/fb0")` and no suite here may touch the live display. **`sd_present_open_fd(fd)`** is now
everything `sd_present_open` does after the open — taking OWNERSHIP of the descriptor —
**`sd_present_open_path(path)`** opens by name and delegates, and `sd_present_open()` is unchanged in
signature, behaviour and cost: `return sd_present_open_path("/dev/fb0");`.
⭐ Gated by the new `programs/present_open_test.cyr` (**131 checks**) over a regular file: the four blocks
refused one at a time, the documented 0 each time, and `/proc/self/fd` counted before and after every
call — 3 descriptors at entry and 3 after each of the nine refusals.
⭐ **It also gates three `memset`s nothing in the tree could see.** Every other caller allocates from the
global bump allocator, which hands back fresh zeroed pages, so deleting `_sd_presenter_build`'s
`memset(blit)` or either of the probe's left all 30 suites green. Group I opens through an arena filled
with `0xAB` and reset — the rewind-and-REUSE shape a consumer actually installs — and MEASURED **171** in
the letterbox, read back out of the file: the previous frame's pixels on their way to the display.
⛔ **What is still ungated is named, not hidden.** Changing `sd_present_open`'s `"/dev/fb0"` to
`"/dev/fb1"`, or replacing its body with `return 0;`, leaves all 30 suites green. That one line is the
residue, and only a display can gate it.

### Verified

All **30** suites pass, including the new `present_open_test` (131 checks); `path_cap_test` 170 → 252 and
`flatten_bound_test` grew its group Q. `fmt --check` clean, `lint` 0 warnings, `vet` clean, `distlib` in
sync with no duplicate top-level names. `sadish_version()` → **800**.
⚠ `SdPolyline` grew 16 → 24 B and `SD_POLYLINE_SIZE` moved with it — the one layout change in this
release; `sd_path_flatten` is the only thing in the ecosystem that allocates one, and its accessors are
unchanged.

### Still open

- `docs/development/proposals/2026-09-15-path-capacity-for-known-size-paths.md` item 3 — **rekha
  adoption**, which this repo cannot verify: `rekha/src/glyf.cyr` still calls `sd_path_new`. rekha's 23
  suites passing against this `dist/` says the change is COMPATIBLE, not that the ask is met.
- Inline `(x, y)` storage in `SdPath` instead of `SdPoint` pointers — **0.9.0**, sequenced on its own
  because it is an ABI break that reaches 14 reads across 6 rekha programs; rekha gets a filing with the
  port and the measured 78,656 → 58,672 B.

## [0.7.2] - 2026-09-16 — the residue a 0.7.1 audit found under four "closed" filings

0.7.1 archived nothing: an audit re-verified every closure claim in `docs/development/` against the
tree and **four of five filings had unmet asks of their own**. This release is that residue. ⛔ Rendering
does not move: agnos's `refagree` prints **BYTE-IDENTICAL on all 200 paths**, and rekha (23 suites) and
dhancha (18) pass against this `dist/`.

### Fixed — a refused allocation inside a STROKE is a return code, not a fault

The last of `docs/development/issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md`.
0.7.1 made path construction, the fill, the clip stack, the presenter and the error record survive a
hook that refuses; strokes still faulted, because `sd_stroke_seg` and `sd_stroke_disc` never tested the
0 that 0.7.1 taught `sd_path_new` to return. `sd_canvas_stroke_path`, `_ex` and `_dash` now return
`SADISH_ERR_OOM` for a refusal at ANY allocation they reach — the per-piece paths and their points, a
curve's flatten mid-points, the scratch and every growth (run, curve flags, batch, dash buffer, the
flush's row accumulator, the area engine's).
⭐ MEASURED, the same calls on 0.7.1 and here (32x32, warm scratch, a hook granting K then refusing):
the round stroker on a line and on a closed 2-cubic blob was **SIGSEGV, rc 139**; the styled and dashed
strokes returned `SADISH_OK` over a fraction of the picture (0 / 6,165 / 12,117 / 18,441 / 28,682 of
59,850 coverage units at K = 0..4) and now return `SADISH_ERR_OOM` **painting exactly the same ink** —
only the report changed.
⚠ **A partial stroke is not rolled back.** What was drawn before the refusal stays; the caller clears
and strokes again, or strokes into a scratch canvas. ⛔ But a refusal leaves no process-lifetime
wreckage: the next stroke on a healthy allocator is byte-identical to a clean one, which rests on
`_sd_sb_flush` emptying the batch BEFORE anything in it can fail — MEASURED as a mutation, a refused
flush that put its count back leaves 270 stale edges and paints 261 of the next stroke's 1,024 pixels
wrong.
⭐ **`src/alloc.cyr`'s seam contract is rewritten.** "A hook returning 0 is handled precisely as
`alloc()` returning 0 already is … No new failure path" was the last sentence standing between the docs
and "a hook may refuse". It now says which entry points return 0, which return `SADISH_ERR_OOM`, what a
refusal costs (nothing for the builders and clip pushes, a partial picture for a fill or stroke), and
what still faults: a caller that ignores a documented 0 — `sd_path_moveto(0, …)`, `sd_polyline_count(0)`,
`sd_canvas_coverage_at(0, …)` are each SIGSEGV, MEASURED.

### Fixed — a fill is ONE flatten operation, not one per curve verb

0.7.1's budget bounded one CURVE inside a fill, so an untrusted outline's work stayed linear in its
curve count — the filing's own severity case, left open. `sd_fill_impl` now opens one operation for the
whole fill, LAZILY at the first curve it really flattens. MEASURED, one unwrapped `sd_canvas_fill_path`
of the hostile 4,096-quad path on 64x64: **16,711,680 B / 1,044,480 allocations → 1,044,480 B / 65,280**
— the same figure as at 1,024 quads — and `sd_flatten_degraded()` now answers for the unwrapped consumer
that used to be told nothing.
⚠ **The trade, taken on evidence and not on taste:** a path whose one fill emits more than
`SD_FLATTEN_BUDGET_DEFAULT` = 65,536 points now degrades its remaining curves to CHORDS mid-fill and
says so. The budget is unchanged because the measurement did not ask for more: the largest legitimate
fill anywhere in this repo or its glyph corpus — all 95 ASCII-shaped glyphs as ONE path at a ~2,048 px
em — emits 26,394 points, **40 % of the budget**; one glyph at that size 960; a 256 px circle of cubics
64. Nothing in the suites, the 200 refagree paths, rekha's 23 suites or dhancha's 18 degrades.
⚠ A fill of a path with NO curve verbs still opens no operation and leaves a consumer's standing verdict
alone — documented since 0.7.1, checked only now.

### Changed — `sd_path_new_cap` sizes the two arrays SEPARATELY

Item 1 of `docs/development/proposals/2026-09-15-path-capacity-for-known-size-paths.md`. 0.7.1 shipped
the two-argument call over ONE capacity field, so `n_points` reached the allocator only through `max()`
and a glyph bought ~2x more verb slots than it uses. `SdPath` now carries two capacities in the SAME
48 B record: `SD_PATH_CAP_OFFSET` (+32) for verbs and the new `SD_PATH_PCAP_OFFSET` (+40) for points —
the word that was `reserved`, written 0 by every constructor and read by nothing (grepped across `src/`,
`programs/`, rekha, dhancha, agnos, and both repos that vendor `dist/sadish.cyr`).
⭐ MEASURED, the proposal's own set (95 ASCII-shaped paths, 1,768 verbs, 2,498 points):

| | bytes | vs `sd_path_new` |
|---|---:|---:|
| `sd_path_new` + pushes | 433,648 | — |
| `sd_path_new_cap`, one capacity (0.7.1) | 84,496 | 5.13x |
| `sd_path_new_cap`, two capacities | **78,656** | **5.51x** |

⇒ the proposal's 78,656 B target to the byte. `sd_path_grow(path, which)` now doubles only the array
that overflowed: a `(8 verbs, 64 points)` path pays **144 B in 2 allocations** for the lineto that
overflows its verbs and keeps its 64 point slots.
⛔ **`SD_PATH_RESERVED_OFFSET` is REMOVED, not aliased, and `sd_path_grow` gained an argument** (no repo
in the ecosystem calls it). New: `sd_path_verb_cap`, `sd_path_point_cap`, `SD_PATH_PCAP_OFFSET`,
`SD_PATH_GROW_VERBS` / `_POINTS`. A record a consumer HAND-BUILT to the 0.7.1 layout carries 0 at +40 and
would read as a zero point capacity — nothing in the ecosystem builds one, and `sd_path_new` /
`sd_path_new_cap` are unchanged in signature and in cost.
⛔ **The verb array is the quiet half.** A growth copies its array's OWN live count; MEASURED with that
one word wrong, **26 of the 28 suites still passed** — a lost verb tag reads back as a valid
`SD_VERB_MOVETO` where a lost point slot is a null `SdPoint` that faults. `path_cap_test` groups E3, E4
and G are the only paths in the suites whose verbs outnumber their points, and they now read every verb
back after a growth.

### Fixed — three coverage loads nothing could tell apart, and three shipped headers that lied

`clip_pitch_test` group K pins the three public coverage loads the 0.7.1 audit found ungated —
`sd_canvas_blit_gradient`'s own load and the `pm != 0` coverage argument in `_sd_paint_blit_run` and
`_sd_paint_blit_point`. No source change: all three were already correct, and each mutation to the
packed index now fails exactly one new check and nothing else, where previously all 27 suites stayed
green.
⚠ Three `@public` headers in `src/path.cyr` stated the opposite of the code and `cyrius distlib` copies
them verbatim into every consumer's bundle: `sd_flatten_op_begin`'s published 16,711,680 B as the
unwrapped cost of the hostile fill (16x wrong after the scope change) and told consumers to wrap a FILL
for a bound they now get by default; it now sells the wrapper for STROKES, which are still per-curve.
`sd_flatten_truncated`'s still called itself "the only witness a starved fill leaves" and said the fill
returns `SADISH_OK` — false since 0.7.1.

### Verified

All **29** suites pass, plus the new `stroke_oom_test` (153 checks) and `path_cap_test`; `clip_pitch_test`
106 → 122 and `flatten_bound_test` 304 → 372. `fmt --check` clean, `lint` 0 warnings, `vet` clean,
`distlib` in sync with no duplicate top-level names. `sadish_version()` → **702**.
⭐ **A review found a gate regression inside this release**: re-measuring three figures for the new fill
scope removed the only check on `sd_flatten_quad`'s own per-curve operation, which the STROKER still
depends on — deleting that operation left all 27 suites green while making identical unwrapped strokes
paint different pictures. `flatten_bound_test` group P is the replacement gate, and it is a leak
detector with absolute ink constants rather than a geometry check.

### Still open

- `docs/development/issues/archived/2026-09-15-flatten-keeps-subdividing-and-allocating-after-the-output-cap-is-full.md` — items 1 and 2 of its Still-open
  section: a truncated contour is still FILLED OPEN rather than refused (there is no flag on
  `SdPolyline`), and a stroke of a starved path reports through the return code but the filing asked for
  the contour itself to be refused or clearly degraded.
- `docs/development/proposals/2026-09-15-path-capacity-…` — items 2 and 3: no in-tree adopter
  (`sd_path_new_cap`'s only callers are two test suites; sadish's own known-size path sites in
  `src/stroke.cyr` still call `sd_path_new`), and inline `(x, y)` storage instead of `SdPoint` pointers,
  which the proposal measures at a further 78,656 → 58,672 B and which is an `SdPath` ABI change.

## [0.7.1] - 2026-09-15 — the repairs rekha filed against 0.7.0

Three filings from the **rekha** team, plus two faults found while closing them. ⛔ Rendering does not
move: agnos's `refagree` prints **BYTE-IDENTICAL on all 200 paths**, the 25 pre-0.7.1 suites pass with
their assertions unedited (one exception, below), and rekha (23 suites) and dhancha (18) pass against
this `dist/`.

### Fixed — `sd_path_flatten` stops allocating mid-points it throws away

Closes `docs/development/issues/archived/2026-09-15-flatten-keeps-subdividing-and-allocating-after-the-output-cap-is-full.md`.
The de Casteljau recursion allocated three `SdPoint`s per quad node (six per cubic) on the seam and kept
only the one that reached the output. It now carries its mids as plain i64 locals and allocates exactly
the points it emits — *(points emitted − 1)* per curve, with no scratch to warm, size or grow.
⭐ **MEASURED, rekha's own repro** (N maximally non-flat quads through `sd_path_flatten`):

| quads | 0.7.0 points / bytes / allocations | 0.7.1 |
|---:|---|---|
| 32 | 8,192 / 457,256 / 24,483 | 8,193 / **327,208** / **8,164** |
| 4,096 | 1,048,577 / 83,623,976 / 3,133,451 | 69,377 / **3,076,136** / **65,287** |

(0.6.0 emitted 8,192 points for 50,200,616 B and silently lost the rest; 0.7.0 kept every point and paid
83.6 MB for them.) A maximally non-flat quad falls from 765 allocations to 255, a depth-8 cubic from
1,386 to 231. The CHANGELOG's one-quad fill costs **48 B** of mid-points where it cost 144 B, and
`alloc_test`'s blob **160 B** where it cost 480 B — every older MEASURED mid-point figure in this repo
read 3x high and has been corrected in place.

⭐ **The recursion also stops when the output is full**, so a curve past the fill-up costs one comparison
instead of a depth-8 subdivision — and the verdict is re-derived per entry rather than remembered,
because raster.cyr and stroke.cyr rewind one per-curve buffer before every verb. ⛔ That second half was
a real defect the review caught before release: without it, one starved fill inside a
`sd_flatten_op_begin`/`_end` scope left **every later fill in that operation returning `SADISH_OK` and
painting a blank canvas**, with the allocator fully restored.

### Added — a per-operation flatten budget: `sd_flatten_budget_set` / `_get`, `sd_flatten_degraded`

`SD_FLATTEN_BUDGET_DEFAULT` = **65,536 points** (0 = unbounded), the shape of 0.7.0's growth ceiling in
the unit flattening allocates in. Over budget the remaining curves degrade to their CHORDS — no
subdivision, no temporaries — and `sd_flatten_degraded()` says so. `sd_flatten_truncated()` reports the
stronger case: points a flatten wanted to emit were LOST.
⚠ **THE BUDGET IS PER OPERATION, AND A FILL OPENS ONE PER CURVE VERB.** A consumer that wants one bound
for a whole draw must scope it. MEASURED, the 4,096-quad path through one `sd_canvas_fill_path` on
64x64 — unwrapped **16,711,680 B in 1,044,480 allocations** (itself a 3x improvement on 0.7.0's
50,135,040 B / 3,133,440); wrapped in `sd_flatten_op_begin()` / `_end()` **1,044,480 B in 65,280**, with
`sd_flatten_degraded()` == 1. ⇒ rekha and dhancha should wrap a glyph draw.

### Added — `sd_path_new_cap(n_verbs, n_points)`

Closes `docs/development/proposals/2026-09-15-path-capacity-for-known-size-paths.md`. An `SdPath` opened
at a caller-known capacity instead of `SD_PATH_CAP` = 256 verbs + 256 points, for consumers that know the
size before the first moveto (rekha converts a glyph outline per glyph, per label, per frame).
⚠ `SdPath` has ONE capacity field for both arrays, so the two arguments collapse to their max, clamped
into [`SD_PATH_CAP_MIN` = 8, `SD_PATH_CAP_MAX` = 2^28]; a capacity above the ceiling is REFUSED (0).
Separate capacities would be an ABI change to the 48 B record, and that decision was left to the owner
rather than taken in a patch release.
⛔ **A capacity could overflow its own byte size**, found while building it: `sd_path_new_cap(2^61 + 1, 0)`
wrapped `cap * 8`, both allocations succeeded, both 0-checks passed, and the result was a live path whose
verb and point blocks were **8 B apart** — the second push wrote off the end. `SD_PATH_CAP_MAX` now bounds
`_sd_path_alloc` before the multiply and `sd_path_grow` before it doubles.

### Fixed — a refused allocation is a return code, not a fault

Closes `docs/development/issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md`.
Every `sd_alloc` / `alloc` result in `path.cyr`, `geom.cyr`, `raster.cyr`, `present.cyr` and `error.cyr`
is checked and propagated: constructors return 0, `sd_path_moveto/lineto/quadto/cubicto` return
`SADISH_ERR_OOM` with the path's verbs, points and contents exactly as they were, and the clip pushes
leave the previous region intact and poppable. On 0.7.0 a hook that returned 0 faulted (SIGSEGV, rc 139).
⛔ **The rule is stronger than "does not crash": a refusal costs the caller nothing it had.**
`sd_clip_push_mask` takes its node BEFORE the in-place mask intersection, so a refusal cannot leave the
caller's shape multiplied by a clip it was never pushed under; `sd_surface_write_ppm` takes both buffers
BEFORE its `O_TRUNC` open, so a refusal no longer truncates an existing file from 35 bytes to 0; the
presenter closes its fd on all four refusal paths.
⭐ **Two sites nobody had filed, both worse than the ones that were.** `_sd_fill_accrow_for` stored the
refused block AND raised the capacity, so one refused request poisoned the fill scratch for the life of
the process and the next fill on any narrower canvas wrote its accumulator through address 0.
`_sd_scratch_init` published a PARTIAL set behind its own "already initialised" flag, so a refusal of the
second of eight blocks meant every later fill stored through a null edge list; it is now all-or-nothing
and the next call retries.
⚠ `sadish_err_new` returning 0 is now a RECORD, not a fault: `sadish_err_code(0)` reads `SADISH_ERR_OOM`.

### Fixed — a starved fill REPORTS instead of painting a wrong picture

⛔ A refused allocation inside a curved fill left the canvas with a fraction of its ink and still returned
`SADISH_OK`. MEASURED: a 4-cubic circle whose true ink is 312,280 coverage units paints **81,029** under a
hook granting 5 allocations. `sd_fill_impl` now scopes the flatten's truncation verdict to the call and
returns `SADISH_ERR_OOM` when that fill lost points; the caller's standing verdict is restored, so nothing
a consumer was already holding is swallowed. ⚠ `programs/flatten_bound_test.cyr` #271 is the one
pre-existing assertion this release edits — it pinned the `SADISH_OK` this fixes, in a suite shipped by
the same release.
⚠ **Strokes are not covered by that return**: the styled path has its own flush and the round stroker
discards `sd_canvas_fill_union`'s result, so `sd_flatten_truncated()` stays their only witness.

### Fixed — a drawing verb before any moveto no longer faults the process

Found while closing the OOM filing, and not in any filing. `cur` is an `SdPoint` POINTER, 0 until the
first moveto, and every drawing verb dereferences it: a path whose first verb is a lineto/quadto/cubicto
was **SIGSEGV, rc 139**, on 0.7.0 and 0.6.0 alike. 0.7.1 made such a path easy to build — a refused
`sd_path_moveto` now returns cleanly, so a caller that logs and carries on has one. Every walk (fill,
round stroke, styled stroke, dash, `sd_path_flatten`) now skips drawing verbs until a moveto arrives,
consuming their points so the verb and point streams stay in step. SVG calls such a path an error and
renders nothing of it; sadish now does the same instead of dying.
⚠ Gated by `programs/integration_test.cyr` group E, where the failure mode is the SUITE faulting rather
than a wrong number.

### Housekeeping — the issue tree, audited rather than tidied

⭐ **Every closure claim in `docs/development/` was re-verified against the tree before anything moved,
and four of the five filings did not survive it.** Only
`2026-09-14-direct-primitives-address-rows-by-width-not-stride.md` is archived (to
`docs/development/issues/archived/`); ten single-site mutations reverting it to the 0.5.5 width-as-pitch
form each fail `stride_test` (5 to 14 checks apiece), and today's suite run against a real 0.5.5 tree
fails **58 of 103** — exactly the figure the filing claimed.
⛔ **The other four stay ACTIVE with corrected status lines and a "Still open" section**, because a
filing closes when its OWN asks are met, not when the headline defect stops reproducing:
- the clip-mask filing: three coverage loads (`sd_canvas_blit_gradient`, and two behind
  `sd_canvas_blit_paint_premul_at`) can still be pushed to the packed index with all 27 suites green —
  the exact gap its closing paragraph claims to have swept for;
- the flatten filing: its "or return an error the fill/stroke can see" is done for the fill and NOT for
  strokes, and a whole untrusted fill is still unbounded unless the caller scopes it;
- the refused-allocation filing: `sd_stroke_seg` and `sd_stroke_disc` still ignore `sd_path_new()`
  returning 0, so a refused allocation inside a stroke faults — the same class the filing is about,
  one level up, and the reason `sd_canvas_stroke_path` cannot yet be called hook-safe;
- the `sd_path_new_cap` proposal: `SdPath` has ONE capacity field, so the two arguments collapse to
  their max and "exactly the given capacity" is not what shipped.
⚠ **A phantom version is gone from the source.** 24 comments in `src/` and `programs/` dated features
to **0.6.1, a release that was never cut** — that work shipped as 0.7.0, and the comments had been
copied verbatim into every `dist/sadish.cyr` since. All 24 now read 0.7.0.
⚠ **File:line citations rot within one release**: one site had drifted 291 lines, and re-pointing an
ellipsis-truncated path in `raster.cyr` shifted 13 freshly-written citations by one line before this
commit landed. The filings now cite FUNCTIONS, with re-derived line numbers kept beside them, and
`docs/development/issues/README.md` (new) makes that a rule along with the grep that proves every
documented path still resolves.

### Verified

All **27** suites pass — 25 pre-0.7.1 with assertions unedited (bar #271 above), plus
`flatten_bound_test` (304 checks) and `oom_test` (157). `fmt --check` clean, `lint` 0 warnings, `vet`
clean, `distlib` in sync with no duplicate top-level names. `sadish_version()` → **701**.
⭐ Mutation-proved as usual, including the two fixes made here: reinstating the fill's `SADISH_OK` fails
`flatten_bound_test` #271, and removing any moveto guard kills `integration_test` with rc 139 — the
failure mode the guard exists to prevent.

## [0.7.0] - 2026-09-15 — dashes, focal gradients, real alpha, and a cap that stops being a cliff

Six items over two waves. ⛔ **Everything a 0.6.0 caller does is byte-identical for every input that
worked in 0.6.0**: agnos's `tests/gpu/refagree.cyr` prints **BYTE-IDENTICAL on all 200 paths** against
this `dist/`, rekha (23 suites) and dhancha (18) pass against it, and the 19 pre-0.7.0 suites pass with
their assertions unedited apart from `area_test` group N, whose 4 assertions pin a limitation this
release deliberately removes.

### Fixed — `SD_FLATTEN_CAP` stops being a cliff: every store that DROPPED data now grows

Through 0.6.0 a path over 8,192 edges filled WRONG (contours left open), a stroke run stopped at point
8,192, `sd_path_flatten` returned 8,192 points, and a styled stroke longer than one batch met itself by
MAX where it crossed. The fill's edge list, the crossings, the stroker's run, the curve flags and the
styled batch now DOUBLE when full — from the GLOBAL `alloc`, never the consumer's hook (0.5.5's rule:
a hook is an arena that gets reset) — and `sd_path_flatten`'s result doubles through `sd_alloc`,
because it is the caller's result on the seam.
⭐ MEASURED, a 20,000-gon disc of radius 120 px on 256x256: 0.6.0 filled **24.5 %** of its ink and left
the centre pixel 0; 0.7.0 fills **99.997 %** of 255·πr² (AREA 99.995 %).
⛔ INITIAL CAPACITIES ARE UNCHANGED, so every first-use figure a consumer has measured is still exact
(fill 590,152 B, styled batch 270,592 B, an AREA styled stroke's first-ever coverage work 860,688 B),
and growth is monotone: a process pays for its largest path once, then **0 B** a call.
⚠ **One styled stroke is now ONE batch** (`SD_STROKE_BATCH_CAP` defaults to 0 = no limit), which
removes the cross-batch MAX loss: MEASURED, 60 random self-crossing walks of 700–1,100 segments —
0.6.0 split 20 (worst 118 levels), 0.7.0 is byte-identical on all 60, and on 60 more of 2,000–5,000
segments where 0.6.0 split every one (worst 131). The flush + re-emission machinery is KEPT as the
fallback for a set cap and for a failed growth. One batch is 5–8 % slower on a big self-crossing
stroke (12,700-edge Lissajous: 41.3 ms vs 39.2 ms) — flushing is faster precisely because each flush
sweeps only its own rows, which is what bought the loss.
⚠ The `SD_AA_AREA` dispatch now falls back on TRUNCATION, not on size, so an AREA fill of exactly
8,192 edges runs the area engine (MEASURED 41 ms, where 0.6.0's fallback took 10.9 s at 8,000 edges).

### Added — `sd_grow_limit_set` / `sd_grow_limit_get`, bounded by default (`SD_GROW_LIMIT_DEFAULT` = 8 MiB)

⛔⛔ **THE TRADE THIS RELEASE MAKES, SAID OUT LOUD.** Lifting the cap turns "a path over 8,192 edges
renders wrong" into "a path over 8,192 edges costs whatever it costs" — and the scratch is
process-lifetime on an allocator with no `free()`, so ONE pathological path makes a process keep that
memory for life. MEASURED on this tree, rekha's hostile repro (4,096 maximally non-flat quads, one
`sd_canvas_fill_path` on 64x64 — the shape a font file can carry): **unbounded**, it grows to
1,048,576 entries and costs **150,602,312 B** of global heap, permanently. Under the default ceiling
the same fill keeps its first 262,144 edges, RECORDS the truncation, and costs **83,493,448 B** — of
which 50,135,040 B is the flatten mid-point waste rekha filed separately (below), not growth.
⇒ Over the ceiling sadish chooses **bounded memory and a flagged, degraded fill** over correct output
at any price; the flags are what keep the area engine off an open contour. `0` restores unbounded
growth; the setter returns the previous value so it can be scoped, like `sd_alloc_set`.
⚠ The ceiling bounds one REQUEST. Doubling abandons the old block, so a store that reaches it has also
paid for every smaller block below (8 MiB + 4 + 2 + … ≈ 16 MiB).

### Added — dashed strokes: `sd_canvas_stroke_path_dash` (new module `src/dash.cyr`)

SVG `stroke-dasharray` / `stroke-dashoffset` over the 0.6.0 styled stroker. `pattern` is a
caller-owned array of `n` 16.16 lengths (on, off, …), `offset` a 16.16 distance into it, any sign. An
ODD `n` is the array read twice; `pattern == 0`, `n <= 0` or an all-zero array is
`sd_canvas_stroke_path_ex` literally; a NEGATIVE entry or an overflowing period is
`SADISH_ERR_BOUNDS` with nothing drawn; the pattern restarts at every subpath.
⭐ **A dash is not a second stroker.** It is `_sd_sb_subpath` run on a PIECE of the flattened run, so
caps at both ends of every dash, joins inside one, the miter limit, the inside-a-curve join rule, the
clip, the `SD_AA_AREA` dispatch and one-call-one-fill all carry over. Both styled entry points now
share one verb walk.
⭐ **A closed subpath dashes around its closing point, and that meeting is a JOIN** — the first dash is
deferred and concatenated onto the last. MEASURED, a 24 px square at width 4, BUTT/MITER, [16,16] at
offset 8: the outer corner reads 255 through the join and 0 with the two dashes stroked apart.
⛔ **A dot is a zero-length ENTRY, not a dash the path ran out of.** Where a boundary falls exactly on
an open subpath's last vertex, the entry has positive length and no path left: nothing is drawn.
`"M 2,4 L 18,4"` under [4,12] is one 4 px dash and a 12 px gap — not a dash and a cap-sized blot at
x = 18. ⚠ Both of these were defects the review caught: the deferred head was stroked through a
recycled cut-point record (a line straight across the shape's interior), and the end-of-path dot
painted a full disc inside a declared gap.
⚠ Dash lengths follow the FLATTENED polyline — a chord-length under-estimate of arc length, so dashes
run long on a curve by about s/(3R): MEASURED at tol = 0.25 px, a radius-20 px circle is 0.63 % short.
Truncation does not creep (`sd_isqrt` floors one-sidedly): 900 raw units over 1,000 diagonal segments
= 0.0137 px, and a full pixel needs 65,536.
**Allocation:** 73,792 B on the first dashed stroke, then 0 B a call. A process that never dashes pays
nothing.

### Added — focal radial gradients and gradient transforms (`src/paint.cyr`)

`sd_gradient_radial_focal(cx, cy, r, fx, fy)` — SVG 1.1 `fx`/`fy` with focal radius 0. A focal point
on or outside the circle is moved inside to `r - ceil(r/2^10)` along its own direction. A focal point
at the centre is `sd_gradient_radial` byte for byte.
⭐ **`t` is EXACT** — `floor(t·2^16)` — for `t < 256`, which is every pixel inside the circle and,
under PAD, every pixel: the root needs ~113 bits, so it is estimated in i64 and the ~2.4 % of pixels
landing near a `2^-16` boundary are settled by the exact sign of the polynomial in 128 bits.
`sd_gradient_set_matrix(g, m)` — SVG `gradientTransform`; `m` maps gradient space to canvas space and
is copied. ⭐ A transformed LINEAR gradient is still a canvas-space linear gradient, resolved once per
blit in 128-bit arithmetic and painted by the same exact DDA, so it costs the same 14.0 ns/px.
⭐ Translations, quarter-turns, 2^k scales and mirrors are byte-identical to moving the geometry;
otherwise MEASURED under `rotate(30°)·scale(2)`, `floor(t·2^16)` is off by 1 at 7 linear / 20 radial /
19 focal pixels of 2,209, never more.
⛔ **Two matrices make the gradient render NOTHING**, both returning `SADISH_ERR_BOUNDS`: a singular
one (SVG's own rule) and one with any entry outside ±16,384.0 (sadish's precision bound, which refuses
perfectly invertible matrices such as a 16,384 px translation). Move the geometry, not the matrix —
and read the return code, because an ignored one leaves an invisible paint.
Also new: `sd_matrix_invert(m)` in `src/geom.cyr` (0 when singular), exact for identity, translations,
2^k scales and quarter-turns. ⚠ `SD_GRADIENT_CAP_OFFSET` is gone — the stop capacity is derived and
`+64` now holds an optional 72 B extension; the object stays 88 B, a plain gradient still costs
exactly 4,256 B, and **a blit still allocates 0 B**, first call included.

### Added — premultiplied writers, and gradient paint into them (new module `src/premul.cyr`)

`sd_clear_premul`, `sd_fill_rect_premul`, `sd_canvas_blit_premul_at` / `_premul`,
`sd_surface_rgba_at`, and `sd_canvas_blit_paint_premul_at` / `_premul`. These are the first sadish
writers whose output has REAL alpha: a transparent clear, a translucent rect, an anti-aliased edge over
a transparent surface, and a gradient panel with AA rounded corners over a transparent window — what a
`SETU_SURF_PREMULTIPLIED` surface needs so agnos composites it with `gpu_shader_op` #92 op 0x01.
⛔ **Alpha is an explicit `a`, and 0 means TRANSPARENT everywhere in the module** — the colour's alpha
byte is never read. #92 reads byte 3 literally, and misreading it is what produced the over-bright
ghosts this stack has paid for twice. Porting a `sd_canvas_blit_at` call means `a = 255`, not
`sd_alpha_of(c)`.
⭐ The arithmetic is agnos's own `cov_ref_px`, with `sd_premul`'s colour. Every output is valid
premultiplied (`c <= a`), swept over all 32,896 valid destination pairs and every (a, cov). Against an
f32 emulation of the kernel shaders: op 0x01 **0 mismatches over 8,421,376 inputs**; op 0x02 worst ±1.
⭐ **ONE EVALUATION, TWO COMPOSITORS**: the straight and premultiplied gradient blits are the same
function with a flag picking the store — same DDA, same roots, same spread, same ramp.
`sd_canvas_blit_paint_at`'s bytes are unchanged, and an OPAQUE gradient at full coverage paints
byte-identical pixels through both.
⚠ With a TRANSLUCENT stop the two roundings compound: MEASURED over ALL 4,261,478,400 lane inputs, the
signed difference premul − straight lies in **−2..+2**, narrowing to −1..+1 at full coverage — not to 0.
⚠ Stop alpha is interpolated STRAIGHT and premultiplied AFTER, per pixel (SVG's rule). A premultiplied
ramp would drag every transition toward the opaque stop: MEASURED, 63 levels at the midpoint.
⚠ A gradient still cannot express a fully transparent stop — the legacy rule spends alpha 0 on OPAQUE.
⚠ Repeated compositing of one translucent layer drifts: alpha stalls below 255 for a < 128, and 20
fills of grey 200 at a = 10 give 98 against an analytic 110.14. A single layer stays within 1.

### Fixed — a clip mask is PACKED `w*h`, on a canvas of any stride

Closes `docs/development/issues/archived/2026-09-15-clip-masks-written-packed-but-read-by-canvas-stride.md`.
The mask was written `y*w + x` by all three producers and read `py*stride + px` by all three consumers
(`sd_fill_impl`, `_sd_area_fill`, `_sd_sb_flush`); they now read it packed. ⛔ Byte-identical — every
canvas `sd_canvas_new` makes has `stride == width`, so no caller can tell.
⚠ **The contract, now stated in the `SdCanvas` layout comment and at each reader:** the COVERAGE buffer
is addressed by the STRIDE and may one day be a window into a wider buffer; the CLIP MASK is a
sadish-owned `w*h` block and is addressed PACKED, always.
Also fixed next door: `sd_canvas_clear` zeroed one flat `w*h` run from the coverage pointer, which on a
wrapped canvas clears neither all of the window nor only the window; it now walks rows at the stride.
Byte-identical on every canvas `sd_canvas_new` makes.

### Filed by rekha — for 0.7.1, not fixed here

- `docs/development/issues/archived/2026-09-15-flatten-keeps-subdividing-and-allocating-after-the-output-cap-is-full.md`
  — `sd_flatten_quad` / `_cubic` keep subdividing (and allocating mid-points) past the output cap.
  ⚠ **This release changes the shape of that cost, and not only for the better.** RE-MEASURED on this
  tree with the same repro: 0.6.0 emitted 8,192 points for 50,200,616 B; 0.7.0 emits all 1,048,577 for
  **83,623,976 B** in **3,133,451** allocations. Truncation is gone; the waste is not. rekha's own note
  anticipated it — "whatever replaces the cap should still bound total flatten work".
- `docs/development/issues/archived/2026-09-15-path-construction-stores-through-a-refused-allocation.md`
  — `sd_path_new` / `sd_point_new` / `sd_path_flatten` store through unchecked `sd_alloc` results, so a
  hook that refuses faults inside sadish (SIGSEGV, rc 139).
- `docs/development/proposals/2026-09-15-path-capacity-for-known-size-paths.md` — `sd_path_new_cap`.

### Verified

All **25** `programs/*_test.cyr` pass: the 19 pre-0.7.0 suites (assertions unedited but for `area_test`
group N) plus `grow_edges_test` (216), `paint_focal_test` (144), `premul_test` (148),
`dash_test` (196), `clip_pitch_test` (106) and `paint_premul_test` (95). `fmt --check` clean, `lint` 0
warnings, `vet` clean, `distlib` in sync with 362 top-level names and none defined twice.
`sadish_version()` → **700**; `dist/sadish.cyr` 170,626 → 297,263 B; DCE smoke binary 16,328 → 16,728 B.
⭐ **Every item was built, adversarially reviewed, fixed and independently re-verified, and each proves
its tests by mutation.** The review found two real dash defects (above) that no suite had caught, and a
clip reviewer found six coverage loads no canvas could reach — correct code that could be pushed to the
wrong index with every suite still green; `clip_pitch_test` groups H and J close them with no source
change. Mutation totals this release: grow 102 (85 caught), paint2 139 (125), premul 72 (63), dash
(groups A–S, every reviewer defect pinned), clip (each reader reverted alone fails only the new suite),
paintpremul (per-pixel clipping pinned against an independent reference after a deleted `+ x0` left all
23 suites green).

## [0.6.0] - 2026-09-15 — the roadmap's three: styled strokes, gradient paint, exact coverage

The three items the README has listed as **next** since 0.4.0 — miter/bevel joins + butt/square
caps, radial + multi-stop gradients, full 2-axis signed-area coverage — plus the open 0.5.5 stride
issue. ⛔ **Everything a 0.5.5 caller already does is byte-identical**: the 14 pre-existing suites
pass with their assertions unedited, agnos's `tests/gpu/refagree.cyr` still prints **BYTE-IDENTICAL on
all 200 paths** against this `dist/`, and rekha (19 suites) and dhancha (18 suites) pass against it.

### Added — `sd_canvas_stroke_path_ex(cv, path, width, cap, join, miter_limit)` (new module `src/stroke.cyr`)

`SD_CAP_BUTT | SD_CAP_ROUND | SD_CAP_SQUARE` × `SD_JOIN_MITER | SD_JOIN_ROUND | SD_JOIN_BEVEL`, and
`miter_limit` as a 16.16 SVG `stroke-miterlimit` ratio (past it a join bevels; AT it, still a miter;
clamped to [1.0, `SD_MITER_LIMIT_MAX` = 1024.0]). An unknown cap or join returns `SADISH_ERR_BOUNDS`
and draws nothing. Open subpaths get caps, closed ones a join at every vertex and no caps; coincident
points are dropped; a lone point draws a disc (ROUND), a 2·hw square (SQUARE) or nothing (BUTT); a
180° reversal is a flat end; a drawing verb after CLOSE restarts at the moveto point, as the fill
reads it.

⛔ **ROUND/ROUND IS `sd_canvas_stroke_path`, UNCHANGED.** The 0.4.0 stroker moved from `raster.cyr`
to `stroke.cyr` verbatim (raster's diff there is a pure deletion) and `_ex(…, ROUND, ROUND, any)`
returns it — asserted against a verbatim 0.5.5 copy across 45 canvases. ⚠ So ROUND/ROUND alone keeps
its 0.4.0 after-CLOSE geometry, per-pass MAX seams, and 34,432 B of paths per closed rect on the seam.

⭐ **Styled pieces are ONE fill, not a MAX union of passes.** A per-piece MAX union is not a union:
two rects meeting at x = 4.5 leave the seam pixel at 127 (MEASURED); as one nonzero fill, 255. Rects,
join triangles, discs and square caps go into a process-lifetime edge batch, all wound the same way,
filled by a stroke-private walk with the default fill's sampling that merges overlapping spans and
bridges 1-LSB touches. Solid pixels reading 254 inside a stroke, fill_union vs the walk: width-8
cubic MITER 120 → 2, ROUND joins 282 → 2, a 10-vertex fractional polyline 36 → 0.
⚠ **Inside one curve's flattening the join style does not apply**: a flattened vertex mitres only
while its tip stays within 0.25 px of the disc, then rounds. At SVG's default limit 4 the cusp of
`M 10,30 C 50,5 10,5 50,30` drew a 6 px spike on a 4 px stroke (MEASURED); now it is round.
⚠ **The limit test is exact integer math** (directions rescaled to [2^13, 2^14), every intermediate
< 2^62); the suite asserts both sides of a ratio of exactly 1.25 and of √298/3.
⚠ **Batches hold 8,192 edges** (~800 miter segments). A longer subpath flushes mid-way and re-emits
the stretch around the seam, so seams ALONG a path stay within 1 level; ⚠ but where a subpath longer
than a batch CROSSES itself, pieces from different batches meet by MAX — MEASURED on 60 random
700–1,100-segment self-crossing walks: 22 split, worst 101 levels.
**Allocation:** 270,592 B once, from the GLOBAL `alloc`, at the first styled stroke (never the hook);
then **0 B** per straight-line styled stroke, hooked or not.

### Added — gradient paint: `SdGradient` + `sd_canvas_blit_paint_at` (new module `src/paint.cyr`)

```
var g = sd_gradient_linear(x0, y0, x1, y1);       # or sd_gradient_radial(cx, cy, r) — 16.16, canvas space
sd_gradient_add_stop(g, 0, sd_rgb(255, 0, 0));     # any order; equal offsets keep insertion order
sd_gradient_add_stop(g, SD_ONE, sd_rgba(0, 0, 255, 128));
sd_gradient_set_spread(g, SD_SPREAD_REFLECT);      # SD_SPREAD_PAD (default) | _REPEAT | _REFLECT
sd_canvas_blit_paint_at(cv, surface, g, dx, dy);   # clips every side, rows by sd_surface_stride
```

⚠ **Sampled at pixel CENTRES** with 16.16 geometry — the legacy `sd_canvas_blit_gradient` samples the
top-left corner with whole-pixel endpoints and is kept byte-identical; they are not interchangeable.
⭐ **t is exact per pixel, not accumulated**: linear t is stepped DDA-style with its remainder, radial
t takes its root to 16 fraction bits. Stops resolve into a 1025-entry ramp, and any ramp cell with a
stop strictly inside it is evaluated from the exact t — so a hard stop lands on the right pixel at any
axis length (a 0.3 stop on a 4,096 px axis flips at pixel 1229) and a 3.5 px band keeps its 4 pixels.
⚠ Stops use the legacy alpha rule (byte 0 = opaque) and interpolate straight alpha; source alpha =
coverage × paint alpha / 255, then `sd_canvas_blit_at`'s src-over. A one-stop opaque gradient paints
exactly `sd_canvas_blit_at`'s bytes. Following SVG: zero stops paint nothing; a zero-length axis or
r <= 0 paints the last stop.
⚠ **Precision bound**: gradient coordinates within ±16,383 px, verified exactly on 4,096 px rows at
the extremes. Outside it colours may be wrong; the ramp index is always clamped.
⭐ **A blit allocates nothing, first call included.** A gradient costs 4,256 B through `sd_alloc`
(object 88 + stops 64 + ramp 4,104); a stop-block doubling adds its new size.
MEASURED, 256×256, ns/px: linear 14.2, radial 44.3 (legacy 2-stop 13.7, solid blit 7.0).

### Added — `SD_AA_AREA`: exact 2-axis coverage (`sd_canvas_set_aa` / `sd_canvas_aa`, new module `src/coverage.cyr`)

The fill has been exact in x since 0.4.0 but SAMPLES y at 4 sub-scanlines: an edge at y = 4.3 reads
**63** (true value 76), at 4.7 **191** (178). `SD_AA_AREA` is signed-area cell accumulation (the
font-rs / stb_truetype v2 / FreeType "smooth" family) and reads 76 and 178.

⛔⛔ **OPT-IN, PER CANVAS; THE DEFAULT IS THE 0.4.0 LOOP, UNTOUCHED** — agnos's GPU rasteriser is gated
byte for byte on it. `sd_canvas_set_aa(cv, SD_AA_AREA)` applies to every fill into that canvas:
`fill_path`, `fill_union` (so the round stroker), the mask `clip_push_path` builds (it inherits the
parent's mode), and — through `_sd_sb_flush`'s dispatch — styled strokes.
⛔ **No streaks.** Every piece deposits exactly its height (`dy - a` into its cell, `a` into the next)
and heights come from exact endpoints, so right of a closed contour the running sum is exactly 0 —
asserted to the raw unit (8,004 edges in one pixel row). ⛔ An AREA fill whose edge list reaches
`SD_FLATTEN_CAP` may have been truncated (an open contour would streak), so it runs the default loop.
⚠ Both rules fold the INTEGRAL of the winding over a pixel: exact where a pixel's winding is {0, ±1},
the standard cell approximation elsewhere (a pixel-sized bow-tie reads 0).
⚠ **`SdCanvas` grew 40 → 48 B** (`SD_CANVAS_AA_OFFSET = 40`); `sd_canvas_new` stores the default
explicitly. Nothing in dhancha, rekha or agnos builds a canvas by hand. ⚠ **Every canvas costs 8 B
more on `sd_alloc`** — dhancha's `text_arena_test` passes unchanged but its printed figures move:
MEASURED against dhancha's current tree, a 12-label frame 415,280 → **415,368 B**, a 10-char label
38,016 → 38,024 B, the 64 KiB-arena spill 349,752 → 349,840 B. Its README/CHANGELOG figures want
re-measuring when it bumps its sadish pin.
MEASURED, 256×256 incl. flattening: curved blob **0.48 ms** per fill (SUBSCANLINE 2.6 ms); 400-edge
self-intersecting polygon 2.2 ms (35 ms); 8,000 edges 41 ms (10.9 s). **Memory:** one (w+1)-slot
accumulator from the global `alloc` on the first AREA fill (264 B at 32 px); **0 B** after.

### Fixed — every surface reader and writer addresses rows by `sd_surface_stride`

Closes `docs/development/issues/archived/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md`.
`sd_plot`, `sd_line`, `sd_hline`, `sd_vline` (and so `sd_rect` / `sd_fill_rect` / `sd_clear`),
`sd_blend_hline` / `sd_fill_rect_blend`, `sd_surface_pixel_at`, `sd_canvas_blit_gradient`, the
presenter's row copy, and `sd_surface_write_ppm` (which the issue missed — it read `w*h` pixels as one
flat run) now load the stride once per call. Packed surfaces are byte-identical (a 66,558 B dump of
every primitive, a PPM and four presenter frames diffs empty against 0.5.5).
⚠ `sd_put(px, w, h, x, y, color)` keeps its signature and stays the PACKED form — a raw pointer
carries no stride; nothing in sadish or its consumers calls it any more. `sd_plot` / `sd_line` use a
private row-pointer store; `sd_line` steps the row pointer rather than re-multiplying (a per-pixel
`px + y*stride` MEASURED ~3.8 % slower; stepped, within noise of 0.5.5).

### Filed

`docs/development/issues/archived/2026-09-15-clip-masks-written-packed-but-read-by-canvas-stride.md` — the
clip mask is written `y*w + x` and read `py*stride + px` by the fill, the area engine and the styled
flush. Unobservable today (every canvas has `stride == w`).

### Changed

- `sadish_version()` → **600**. `dist/sadish.cyr` 85,815 → 170,626 B; DCE smoke binary 16,000 → 16,328 B.
- Module order: `coverage.cyr` before `raster.cyr`; `stroke.cyr`, then `paint.cyr`, after it.

### Verified

All **19** `programs/*_test.cyr` pass — the 14 pre-existing suites with assertions unedited, plus
`stride_test` (103 checks; 58 fail against the 0.5.5 sources), `stroke_style_test` (193),
`paint_test` (248), `area_test` (163) and `integration_test` (33: the process's first-ever coverage
work is an AREA styled stroke under an arena hook — exactly 860,688 B on the global heap, **0** on
the arena, 0 / 0 warm; styled strokes on an AREA canvas read 51 / 25 where the walk reads 63 / 31;
clip + MAX through that dispatch; paint through AREA coverage onto a wrapped surface equals
`sd_canvas_blit_at`; the version probe). `fmt --check` clean, `lint` 0 warnings, `vet` clean,
`distlib` in sync. No new top-level name collides with anything in dhancha, rekha, crab or
agnos/tests/gpu.
⭐ **Every item was built, adversarially reviewed, fixed and independently re-verified, and each
proves its tests by mutation**: stride 22 mutations, all caught; stroke 103 (89 caught, 3 kill the
suite, 11 equivalent); paint 114 (110 caught, 4 equivalent); area 75 (68 caught, 7 equivalent or
speed-only); integration 3, all caught (no AREA dispatch: 8 checks; its guard inverted: 2; the
accumulator on the hook: 2).
⚠ **One pre-release rename**: the stroke item defined both `var _sd_sb_cap` (batch capacity) and
`fn _sd_sb_cap` (draw a cap) — the fn is now `_sd_sb_endcap`. Duplicate names shadow silently here.

## [0.5.5] - 2026-09-14 — a rasterizer that can be told where its memory comes from

### Added — `sd_alloc` / `sd_alloc_set` / `sd_alloc_get` (new first module `src/alloc.cyr`)

⭐⭐ **The allocation seam.** 23 of the 32 `alloc(` sites sadish had (error 1, geom 2, path 8,
present 6, surface 2, raster 4) now go through `sd_alloc(n)`; the other 9 — `sd_fill_impl`'s
ectx/edges/fbuf/fctx/cross/accrow and the stroker's run/fbuf/fctx — became the process-lifetime
scratch in `_sd_scratch_init` / `_sd_fill_accrow_for`, deliberately on the global `alloc` (⛔
below). A consumer may install a hook:

```
var prev = sd_alloc_set(&my_arena_hook);   # fn(n): ptr, 0 on failure; returns the PREVIOUS hook
... draw ...
sd_alloc_set(prev);                        # 0 restores the global allocator; sd_alloc_get() reads it
```

`lib/alloc.cyr` is a bump allocator with **no `free()`**, so until now every canvas, path, point
and clip mask sadish made was permanent. dhancha's frame arena (`dh_falloc`) is the only way a
rendered frame can cost the heap nothing, and dhancha could not route sadish onto it — the defect
it filed as `2026-09-13-scalable-text-allocates-per-call-outside-the-frame-arena.md`, and one of
the two blockers on **crab** adopting a proportional face (crab's headline gate asserts a rendered
frame costs the global heap **exactly 0 bytes**). This is the sadish half of closing it.

⚠ **The hook applies to EVERYTHING sadish allocates while it is set** — surfaces, canvases,
coverage buffers, clip masks, paths, verb/point arrays, points, matrices, flatten mid-points, the
PPM/presenter buffers. sadish does not distinguish the result from the working set; the consumer
scopes the hook around the work whose lifetime it is choosing. ⇒ Nothing returned under an arena
hook may be kept across that arena's reset.
⚠ A hook returning 0 is handled exactly as `alloc()` returning 0 already was. No new failure path.

### Added — `sd_canvas_blit_at(cv, surface, color, dx, dy)`

Composite the canvas with its (0,0) at surface pixel `(dx, dy)`; either may be negative or beyond
the surface, and the blit clips to the surface on every side. Same src-over math as
`sd_canvas_blit`, verbatim — which is now `sd_canvas_blit_at(cv, s, c, 0, 0)`.

⚠ **Rows are addressed by `sd_surface_stride`, not `width*4`.** The pre-0.5.5 `sd_canvas_blit`
used the surface WIDTH as its row pitch — right for every packed surface sadish makes itself, and
one row of shear per row on a WRAPPED surface (dhancha's `dh_surface_wrap` over a framebuffer or a
pane sub-rect, where `stride != width*4`). The suite builds such a header by hand and asserts row
1 lands at the pitch (128 B), not at `width*4` (32 B).
⚠ **This is now the ONE writer that honours the stride.** `sd_put`, `sd_surface_pixel_at`,
`sd_hline`, `sd_vline`, `sd_blend_hline`, `sd_canvas_blit_gradient` and the presenter's row copy still
address by `width * 4` — right for every packed surface sadish makes, one row of shear per row on a
wrapped one, MEASURED as 4,000 padding pixels overwritten by a `WINDOW` background under text that
landed straight. Filed as
`docs/development/issues/archived/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md`.

### Changed — the fill/stroke scratch is process-lifetime, allocated once

⭐ `sd_fill_impl` allocated `ectx + edges + fbuf + fctx + accrow + cross` FRESH on every call, and
`sd_canvas_stroke_path` its `run + fbuf + fctx` — all sized to the fixed `SD_FLATTEN_CAP`, none of
it dependent on the path. **MEASURED on 0.5.4** (8x8 canvas, one call, `alloc_used()` delta):

| call | 0.5.4, every call | 0.5.5, first call | 0.5.5, every call after |
|---|---|---|---|
| `sd_canvas_fill_path`, 4-line rect | 327,824 B | 589,960 B | **0 B** |
| `sd_canvas_fill_path`, 1 quad + lines | 328,016 B | — | 144 B (9 flatten mid-points) |
| `sd_canvas_fill_path`, 760x300 triangle | 333,856 B | 6,080 B (accrow) | **0 B** |
| `sd_canvas_stroke_path`, closed 4-vertex rect | 2,789,016 B | — | 34,432 B (8 paths + points) |
| one frame of fill+union+stroke+clip+blit (16x16) | 3,456,048 B | — | **0 B** under an arena hook (45,000 B on the arena) |

The scratch is now one process-wide set — `_sd_fill_*` and `_sd_stroke_*` module globals in
`raster.cyr`, allocated lazily ONCE by `_sd_scratch_init()` (MEASURED 589,896 B: fill 458,800 +
stroke 131,096) plus a per-row accumulator that re-allocates only when a WIDER canvas than ever
seen arrives (w*8: 64 B at 8 px, 6,080 B at 760). Every per-call reset (edge count, flatten ctx)
is exactly as before, and the output is **byte-identical**: all 13 pre-existing suites pass
unchanged, and a probe dumping coverage for a rect, a quad, a cubic and a stroke diffs empty
against 0.5.4.

⛔ **From the GLOBAL `alloc`, never the hook.** A consumer's hook is typically an arena that gets
reset every frame; a scratch drawn from it would be dangling on the second frame. So the scratch
is the one thing sadish allocates that the seam does not see — its one-time cost lands on the
global heap at the first fill, and never again.
⛔ **Two sets, not one.** `sd_canvas_stroke_path` holds `run` live across the `sd_canvas_fill_union`
calls `sd_stroke_run` makes; the stroker's `run/fbuf/fctx` are its own globals so non-aliasing is
visible in the names rather than reasoned about per call.
⚠ **Single-threaded entry points, said out loud.** The rasterizer never had a lock anywhere and was
never thread-safe; a shared scratch makes `sd_fill_impl` / `sd_canvas_stroke_path` explicitly so.
Checked: every consumer (dhancha, rekha, agnos's refagree test) rasterizes from one thread.
The flatten MID-POINTS stay on the seam: proportional to the curve work, not fixed-capacity, so the
hook decides where they live.

### Changed — toolchain `6.6.2` → `6.6.4`

Bumped before any source change; all 13 suites passed on the new pin first. `sadish_version()`
now answers **505** (it had read 401 since 0.4.1). `dist/sadish.cyr` 75,450 → 85,815 B; the DCE
smoke binary 15,912 → 16,000 B.

### Verified

All **14** `programs/*_test.cyr` pass (13 existing + the new `alloc_test`, 96 numbered checks).
`fmt --check` clean, `lint` 0 warnings, `vet` clean, `distlib` in sync.
⛔ **The process's FIRST fill runs under an arena hook** (`alloc_test` group A). With no hook
installed `sd_alloc` IS `alloc`, so a warm-up fill outside a hook lands the scratch on the global
heap under either spelling and cannot see the ⛔ above: an earlier draft of the suite warmed up
unhooked, and all nine scratch sites flipped `alloc` → `sd_alloc` passed it 90/90. The gate is now
the pair of exact figures — 590,152 B (589,896 scratch + 32*8 accrow) on the global heap and
EXACTLY 0 on the arena — followed by a wider (64 px) canvas under the same hook whose accrow
re-grow is exactly 512 B on the heap and 0 on the arena.
⭐ **Fifteen mutations, each of which fails the suite** (failed checks in brackets): a raster
`sd_alloc(` back to `alloc(` for the coverage buffer [2] and for the clip node [2]; `sd_point_new`
off the seam [3]; fill edges + cross back to per-call `alloc` [8]; the stroke run per-call [3];
the accrow capacity never remembered [3]; `blit_at` ignoring `dy` [8]; `blit_at` using `width*4`
as the pitch [4]; `blit_at` not clipping on the left [2, then the process faults — a mis-clipped
column writes into the surface header's stride field]; not clipping on the right [3];
`sd_alloc_set` returning the new hook instead of the previous [8]; `sd_alloc` ignoring the hook
[4]; all nine scratch sites `alloc` → `sd_alloc` [4, then the process faults in group B — the
scratch dangles after the first `arena_reset`, which is the corruption the ⛔ exists to prevent];
the accrow site alone [4]; the eight fixed-scratch sites alone [2, then faults].

## [0.5.4] - 2026-09-11

### Changed

- **Toolchain `6.5.36` → `6.6.2`.** No source change. Zero compiler rejections,
  zero fail-open sites. All `programs/*_test.cyr` RUN suites pass.

## [0.5.3] - 2026-08-31 — a fill that reads its destination

### Added — `sd_fill_rect_blend` / `sd_blend_hline`

⭐⭐ **The first fill in sadish that READS its destination.** Every other span writer stores four
bytes per pixel and never loads — correct and fast for opaque paint, and exactly why a translucent
veil was impossible. dhancha wanted a modal backdrop for its 0.9.23 sheet and had to ship a
**scanline dither** instead; this is what that was waiting on.

```
sd_fill_rect_blend(s, x, y, w, h, color, a)     # a = 0..255 coverage
sd_blend_hline(s, x0, x1, y, color, a)          # the span workhorse
```

⛔⛔ **`a` IS AN EXPLICIT ARGUMENT AND 0 MEANS FULLY TRANSPARENT — THE OPPOSITE OF `sd_alpha_of`.**
The legacy packed-alpha rule maps a 0 byte to **opaque**, which exists so a bare `sd_rgb` value is
not invisible. Carrying that convention into a blend would make `sd_fill_rect_blend(..., 0)` paint
**solid** — the single most surprising outcome available, and the exact trap that made
`sd_rgba(0, 0, 0, 128)` paint black rather than a half veil. ⇒ Alpha is passed separately, it is
never read out of `color`, and 0 is a no-op. The test asserts that in both directions.

⚠ **Source-over, integer, per channel**: `out = (src*a + dst*(255-a)) / 255`. Rounding truncates, so
two 50 % veils are not identical to one at 75 % — true of any integer compositor and not worth a
fractional pixel format to fix.
⚠ **The destination alpha byte is left at 255.** sadish surfaces are opaque render targets; a blend
that also composited alpha would make the result translucent to whatever draws it next, and every
consumer here presents to a screen.

### Changed — toolchain pin 6.5.27 → 6.5.36

⚠ **Nine releases stale.** Per the standing rule a repaired repo does not stay on an old pin; `lib/`
re-vendored with `cyrius lib sync`.

### Verified

All **14** `programs/*_test.cyr` pass (13 existing + the new `blend_test`). `fmt --check` clean,
`lint` 0 warnings.
⭐ **Three mutations, each of which fails the suite**: `a == 0` treated as opaque (the legacy trap);
the blend ignoring its destination and storing the source directly; and the red channel reading the
blue source, which only shows on a non-grey blend.

## [0.5.2] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.5"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **THE ARTIFACT CHANGED, so this is not a cosmetic edit.** The build went `118032 -> 126448` bytes and the binary differs. The pin is not a comment: it selects the stdlib snapshot under `~/.cyrius/versions/<pin>/lib`, so moving it swaps the library code this repo compiles against.

⚠ The vendored `lib/` was re-synced to the 6.5.27 bundled set, which clears the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [0.5.1] - 2026-08-02

### Changed — cyrius pin 6.4.71 -> 6.5.5

Toolchain catch-up across the whole desktop stack, cut together so the next burn runs binaries built
by ONE compiler rather than 6 different ones.

⚠ **The pin was documentation, not enforcement.** `cyrius build` compiles with the INSTALLED `cycc`,
prints a `toolchain drift` warning, and carries on — so this project was already being built by 6.5.5
before this bump. Verify provenance with `~/.cyrius/versions/<pin>/bin/cyrius` when it matters.

⭐ What the gap actually contained, for a reader deciding whether to care:
- **6.5.1** made overload-suffix arity a hard **error** where it used to warn. Latent arity
  mismatches are now build failures instead of silently-wrong code — good, and the reason this
  sweep surfaced real defects elsewhere in the stack.
- **6.4.75** fixed `fn_table` growth past 8192 silently corrupting six fn-indexed side tables.
- **6.5.0** added file-scoped `private` / per-item `public` — the first real answer to this
  ecosystem's duplicate-`fn`-silently-shadows hazard.
- **6.4.82** completed the agnos GPU syscall wrapper band to `#82`-`#95`, so `sys_gpu_shader_op`
  (#92) and `sys_gpu_modeset_op` (#93) no longer need a raw `syscall()` behind an `#ifdef`.

### Verification

Host + `--agnos` builds green; **12 RUN tests** pass; `distlib` regenerated.

## [0.5.0] - 2026-07-23

### Added — a real ALPHA channel: `sd_rgba` · `sd_alpha_of` · `sd_color_a` · `sd_premul`

Until now every primitive hardcoded the alpha byte to **255 in four separate places** and the colour type
had no alpha at all. Invisible, because nothing downstream read byte 3 — but agnos's `gpu_shader_op` **#92**
op 0x01 does: it performs **premultiplied src-over on the GPU shader cores**, the one thing a CPU blit
cannot do cheaply. Without a real alpha channel that whole hardware-blend path had **no producer**.

`sd_put`, `sd_hline` and `sd_vline` now write the colour's alpha instead of a hardcoded 255. (`sd_fill_rect`
and `sd_clear` route through `sd_hline`, so fixing `sd_put` alone would not have covered fills — the three
sites were independent.)

⚠ **An alpha byte of 0 means OPAQUE, not invisible, and that is what makes this additive.** `sd_rgb(r,g,b)`
returns `0x00RRGGBB` with byte 3 = 0, and every existing caller and test depends on that value; treating 0
as transparent would make every pre-existing drawing vanish. `sd_alpha_of()` maps 0 → 255. The cost is that
fully-transparent is not expressible as a colour — the right trade, since it is the one alpha value with no
visible effect.

⚠ `sd_premul()` is the **sanctioned producer** for surfaces flagged `SETU_SURF_PREMULTIPLIED`. Passing
straight alpha to #92 does not error — it renders washed out, silently, with no diagnostic anywhere.

**Not changed:** `sd_canvas_blit`'s coverage path still flattens anti-aliased coverage into opaque RGB. That
is correct for an opaque destination and is not what blocked #92; a premultiplied coverage path would be a
new function, not a change to this one.

## [0.4.2] - 2026-07-23

### Changed — cyrius pin 6.4.25 → 6.4.71

Toolchain refresh across the draw stack. Materialised `lib/` re-synced (`cyrius lib sync --full`).
No source change; build + tests green at the new pin.

## [0.4.1] - 2026-07-08 — toolchain alignment

Pin/hygiene release — no code change; the 2D vector core is byte-identical to
0.4.0 (all RUN tests green).

### Changed

- **Cyrius pin `6.4.7` → `6.4.25`** — aligns sadish with the desktop stack
  (setu + dhancha, its consumer, both pin `6.4.25`) instead of drifting behind.
  Builds + all 13 RUN tests (aa / blit / clip / draw / fill / flatten / geom /
  gradient / grow / present / rotate / stroke / smoke) pass.
- `sadish_version()` → **401**.

## [0.4.0] - 2026-07-05

The full 2D vector core: skew (affine set complete), gradient paint, unbounded
paths, stroking, a clip stack, and an analytic coverage engine replacing the
supersampler. 12 RUN tests.

### Added
- **Matrix skew** (`sd_matrix_skew`, `sd_fixed_div`) — completes the affine
  transform set (identity / translate / scale / rotate / skew); shear angles
  via CORDIC `tan = sin/cos`. Verified in `programs/rotate_test.cyr`.
- **Linear gradient paint** (`sd_canvas_blit_gradient`) — composites coverage
  with the source color lerped along a linear axis from `c0` to `c1`
  (projection clamped to the endpoints), src-over onto an `SdSurface`.
  Verified in `programs/gradient_test.cyr`.
- **Growable path backing** (`sd_path_grow`) — `SdPath` is now unbounded: the
  verb + point arrays double on overflow instead of capping at
  `SD_PATH_CAP` (256). Verified in `programs/grow_test.cyr` (1000 verbs across
  several grows, data preserved).
- **Stroking** (`sd_canvas_stroke_path`, `sd_isqrt`, `sd_canvas_fill_union`) —
  strokes a path with round caps + round joins: a rectangle per flattened
  segment (perpendicular offset via integer sqrt) + a disc per vertex,
  MAX-unioned into the coverage so overlaps combine regardless of winding.
  Curves flatten first; subpaths + close are honored. Verified in
  `programs/stroke_test.cyr`. (Miter/bevel joins + butt/square caps + piece
  batching are TODO(v0.5).)
- **Clip paths / clip stack** (`sd_canvas_clip_push_rect` /
  `sd_canvas_clip_push_path` / `sd_canvas_clip_pop`) — a coverage-mask clip
  region held as a stack on the `SdCanvas`; push intersects a rect or a filled
  path into the active clip, fills + strokes are multiplied by the mask, pop
  restores. Verified in `programs/clip_test.cyr` (nested intersect + pop).

### Changed
- **Analytic coverage rasterizer** — the fill core (`sd_fill_impl`) now
  computes coverage ANALYTICALLY in x (exact horizontal span overlap via
  `sd_span_add`) across `SD_FILL_SS` vertical sub-scanlines, replacing the 4×4
  point supersampling. Smoother sub-pixel AA (no longer quantized to 17 levels)
  and much cheaper — edge crossings are computed per row, not per pixel. Every
  fill / stroke / clip inherits it; the whole suite stays green. Verified in
  `programs/aa_test.cyr` (0.3 and 0.7 edge coverage → 76 / 178, which the
  supersampler could not produce). Full 2-axis signed-area accumulation is
  TODO(v0.5).

## [0.3.0] - 2026-07-05

The rasterizer — the vector core comes alive. Paths now flatten, fill to
anti-aliased coverage, and composite onto a surface; plus affine rotation.
All five pieces are RUN-tested.

### Added
- **Signed fixed-point** (`sd_asr` + `sd_fixed_mul` routed through it, plus
  `sd_abs`) — Cyrius `>>` is logical (zero-fill); the signed arithmetic-right
  shift keeps negative products (mirrors, rotations, below-origin coords)
  correct. Guarded by `programs/geom_test.cyr`.
- **Adaptive Bézier flattening** (`sd_path_flatten`) — real de Casteljau
  subdivision for quad + cubic verbs, second-difference flatness vs `tol`,
  midpoints via `sd_asr`. Replaces the anchor-only stub.
  (`programs/flatten_test.cyr`.)
- **Scanline coverage fill** (`sd_canvas_fill_path`) — the headline: builds a
  closed-edge list from the flattened path (each subpath closed back to its
  moveto), then supersampled (4×4) anti-aliased coverage via a +x winding ray
  per sub-sample, honoring both nonzero and even-odd rules.
  (`programs/fill_test.cyr`.)
- **Coverage composite** (`sd_canvas_blit`) — retyped from the raw-`fb` stub to
  composite coverage × a solid color, src-over, onto an `SdSurface`.
  (`programs/blit_test.cyr`.)
- **Affine rotate** (`sd_matrix_rotate`, `sd_cos`/`sd_sin`) — sovereign
  fixed-point trig via CORDIC (16 iterations, shifts + adds only, no float / no
  libm), range-reduced to [-π/2, π/2]. (`programs/rotate_test.cyr`.)

### Notes
- Coverage is supersampled (4×4 → 17 levels); an analytic signed-area cell
  accumulator (smoother AA, lower cost) is TODO(v0.4), along with gradient
  paints, clip paths, stroking, matrix skew, and a growable path/flatten
  backing.

## [0.2.0] - 2026-07-05

The "easy 80%" lift — sadish stops being a scaffold. A real pixel surface,
the direct 2D primitives, and a present backend, all **adapted from the
ecosystem's framebuffer games rather than reinvented** (see
`docs/development/prior-art.md`), and all CI-gated. The anti-aliased coverage
rasterizer + adaptive Bézier flattening land in v0.3.0.

### Added
- **Pixel surface + direct primitives** (`src/surface.cyr`, `src/draw.cyr`) —
  `SdSurface`, a 32bpp BGRA pixel buffer (packed `0x00RRGGBB` colors, top-left
  y-down, `off=(y*w+x)*4`), plus the opaque integer-pixel primitives
  `sd_plot` / `sd_line` (integer Bresenham) / `sd_hline` / `sd_vline` /
  `sd_rect` / `sd_fill_rect` / `sd_clear`, and `sd_rgb` + channel extractors.
  Lifted from the ecosystem's framebuffer games (encom-hits `draw.cyr`
  Bresenham; the BGRA/present convention shared with cyrius-bb/-polyomino/
  -doom). Verified by `programs/draw_test.cyr`, a headless pixel-readback RUN
  test (exact-diagonal Bresenham, span clamping, filled vs hollow rect, silent
  out-of-bounds clip).
- **Present backend** (`src/present.cyr`) — `sd_surface_write_ppm` (binary P6
  PPM dump, platform-neutral file syscalls) + the Linux `/dev/fb0` `SdPresenter`
  (`sd_present_open`/`_blit`/`_close`: geometry probe, largest-integer-scale
  center-letterbox blit honoring the physical pitch, 32bpp + RGB565 paths),
  adapted from encom-hits' `engine.cyr`. Struct-based/reentrant (the encom
  original was module-global + fixed 320×240). Verified by
  `programs/present_test.cyr` (PPM write + readback); the fb0 path is never
  exercised in tests — it writes the live display. The AGNOS `blit`#39 sink is
  a future backend beside fb0.
- **CI + release** (`.github/workflows/{ci,release}.yml`) — build / lint / fmt /
  vet / dist-sync + RUN-test suites, a security scan (raw execve/fork/sys_system),
  and docs + version-consistency gates; semver-tag release that archives the src
  tarball + `dist/sadish.cyr` + `SHA256SUMS`. Toolchain pinned via
  `cyrius.cyml` (no version hardcoded in YAML).

## [0.1.0] - 2026-07-05

### Added
- Repo scaffolded: a buildable, link-clean pure-Cyrius skeleton for the
  low-level 2D vector graphics core — `SdPoint`/`SdMatrix` (16.16
  fixed-point geometry, live), `SdPath` (verb + point construction, live)
  with a `sd_path_flatten` stub, `SdCanvas` (coverage buffer) with
  `sd_canvas_fill_path`/`_blit` stubs, the `SadishErr` model, and a
  `programs/smoke.cyr` link-check. The scanline coverage rasterizer +
  adaptive Bézier flattening land in v0.3.0.
