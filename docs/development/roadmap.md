# sadish roadmap

**Where we are:** 0.11.0. 34 RUN suites green, `fmt`/`lint`/`vet` clean, `dist/sadish.cyr` in
sync, toolchain pinned to **6.6.6**, and the host build is diagnostic-free. 0.10.0 took the
`SdPolyline` ABI break this file filed under "What 1.0 freezes", so the library now stores points
ONE way; a fill and a styled or dashed stroke allocate NOTHING on the consumer's seam. agnos's `tests/gpu/refagree.cyr` holds the default fill byte-identical
across 200 random paths — it has stayed green through every release since 0.6.0 and is the standing
proof that a change did not move a pixel.

Each entry below says how its claim was established: **VERIFIED** = read out of the tree while writing
this, **MEASURED** = a figure from the release that shipped it, **UNKNOWN** = cannot be settled from
inside this repo.

## Shipped

The measurements and contracts live in [`../../CHANGELOG.md`](../../CHANGELOG.md); this is the index.

| | |
|---|---|
| v0.2.0 | `SdSurface` + the opaque integer primitives + the present backend |
| v0.3.0 | the rasterizer: flattening, scanline AA coverage, src-over blit, CORDIC rotate |
| v0.4.0 | the full 2D core: skew, gradients, growable paths, stroking, clip stack, analytic-x coverage |
| v0.5.0–0.5.5 | a real alpha channel, `sd_fill_rect_blend`, and the `sd_alloc` seam |
| v0.6.0 | styled strokes (caps/joins/miter), `SdGradient` paint, opt-in `SD_AA_AREA`, stride-correct surfaces |
| v0.7.0 | dashes, focal radial + gradient transforms, premultiplied writers, growable edge storage |
| v0.7.1–0.7.2 | bounded flattening, checked allocations, hook-safe strokes |
| v0.8.0 | exact-size piece paths, a self-describing `SdPolyline`, a testable presenter |
| v0.9.0 | `SdPath` stores its points inline — the ASCII glyph set 78,656 → 58,672 B, 2,783 → 285 allocations |
| v0.9.1 | the 6.6.6 pin; a format gate that actually gates; portable syscall constants + an aarch64/AGNOS cross-build |
| v0.10.0 | `SdPolyline` points inline — a flatten is 5 allocations, not 20,004; `sd_path_transform`; `sd_path_bounds` |
| v0.11.0 | pattern paint + `SD_SPREAD_NONE`; `docs/api.md`; the `/dev/fb0` device line gated |

## The road to 1.0

⚠ **1.0 is a promise not to break the ABI**, and 0.9.0 broke it three weeks into the project's life.
Everything under "What 1.0 freezes" should be settled BEFORE the number is cut, because after it they
are permanent.

### Capability gaps

**~~Image / pattern paint.~~ CLOSED in 0.11.0** — `sd_pattern_new(src)` is a fourth paint kind
(`SD_PAINT_PATTERN`) beside the three gradients, sharing the record, the spread, the matrix and both
blit entries. The blast radius was exactly as predicted: a kind plus a sampler, and the coverage
engine never moved.
⭐ Cheaper than the entry assumed: a pattern is **one 88 B allocation** (no stops, no ramp) and a
pattern blit allocates **nothing**, on both the plain and matrix-mapped paths.
⚠ **The two decisions, as taken.** Filtering is **nearest only**, but through a FIELD —
`sd_pattern_set_filter` answers `SADISH_ERR_UNSUPPORTED` for anything else, so bilinear lands later
without an ABI break and a consumer is never silently given a filter it did not ask for. Outside the
rect reuses the gradient spreads and adds `SD_SPREAD_NONE` ("paint nothing"), which SKIPS a pixel
rather than writing it transparent — the straight blit forces dst alpha 255, so those are different
pictures. ⛔ A gradient may not have `SD_SPREAD_NONE`: it is the library's first ever use of
`SADISH_ERR_UNSUPPORTED`.
⚠ **Still open, deliberately:** bilinear itself, and the question of whether a pattern should ever
carry its own tile rect rather than taking the whole source surface.

**Arcs.** VERIFIED: the verb set is `SD_VERB_MOVETO / LINETO / QUADTO / CUBICTO / CLOSE`. SVG's `A`
and every rounded rectangle are hand-rolled by the caller into cubics today. Blast radius: a verb tag
plus a flattener; the verb/point stream pairing is already per-verb, and a new verb consumes its own
point count. ⚠ The alternative is to declare arcs a CONSUMER concern and document the cubic
approximation as the supported path — cheaper, and defensible for a core this size.

**~~`sd_path_transform`.~~ CLOSED in 0.10.0** — `sd_path_transform(path, m)` affines every point in
place and allocates nothing, because 0.9.0's inline slots made it a read, six multiplies and a write.
MEASURED against the loop it replaces on a 7-point path: the hand-rolled `sd_matrix_apply` rebuild
costs 14 `sd_alloc` calls and 224 B, this costs 0 and 0. Gated point-by-point against
`sd_matrix_apply` across five matrices (`programs/transform_test.cyr` group D).

**~~Path bounds.~~ CLOSED in 0.10.0** — `sd_path_bounds(path, out)` writes four 16.16 words to a
caller's 32 B buffer. ⚠ RECOMPUTED, not cached, exactly as this entry predicted: the 48 B record has
no spare word, so caching would be an ABI break of its own and was not taken.
⚠ It is the CONTROL-POINT hull, so a path with curves reports a box that contains the drawn shape and
may exceed it — the conservative answer culling and damage tracking want, gated as a real superset
(`programs/transform_test.cyr` group H). An empty path is `SADISH_ERR_EMPTY_PATH` with `out`
untouched, so a degenerate box at the origin can never be mistaken for a real one.

**Blend modes.** VERIFIED: every compositor in the tree is src-over (`src/premul.cyr`, `src/paint.cyr`,
`src/raster.cyr`). No multiply/screen/darken. ⚠ UNKNOWN whether the consumers want them: that is a
dhancha/crab question, not a sadish one, and the answer decides whether this belongs in 1.0 at all.

### What 1.0 freezes

**~~The `SdPolyline` / `SdPath` split.~~ CLOSED in 0.10.0**, and it was the right call to take it
before 1.0 rather than after. `SdPolyline`'s points are inline 16 B slots like `SdPath`'s, and
`sd_polyline_point_x` / `_y` are the accessors — the pattern copied from 0.9.0 exactly as this entry
proposed. ⭐ The deferral had a measurable price that is now repaid: while the output held pointers,
the flatten had to MATERIALISE a record per emitted point, so what 0.9.0 stopped paying per point the
flatten started paying. A fill of the hostile 4,096-quad path went 1,110,016 B / 69,376 calls → **0 /
0**. ⚠ `SdPoint` itself stays published as an argument type (`sd_matrix_apply` takes and returns
one), but no STORE in sadish is an `SdPoint` any more.

**The error model.** VERIFIED: `src/error.cyr` defines `SadishErr` with codes and a detail pointer, and
almost nothing constructs one — the library returns bare `SADISH_*` codes and 0. Either the record is
the contract or the codes are; shipping 1.0 with both leaves a half-built API permanently.

**What a hook can refuse — narrowed by 0.10.0, and worth settling before 1.0.** VERIFIED: a fill, and
a styled or dashed stroke, no longer allocate on the `sd_alloc` seam at all, so a consumer's hook
cannot starve them. Their `SADISH_ERR_OOM` returns are still reachable — through the process scratch
and the edge list — but only from the GLOBAL allocator, which a hook never controlled. ⚠ The ROUND
stroker is the exception and still allocates per piece. ⇒ The documented contract ("a refusal is a
return code at every public entry point that can reach one") is still true and now covers far fewer
situations. Before 1.0, decide whether that asymmetry is the shipped contract or whether the round
stroker should be brought in line; `src/alloc.cyr` states it either way and
`programs/stroke_oom_test.cyr` gates both halves.

### Infrastructure

**~~No API reference.~~ CLOSED in 0.11.0** — [`../api.md`](../api.md) covers all 122 public
functions by module and front-loads the six cross-cutting rules most consumer bugs come from (16.16
and the logical `>>`; the allocation seam; what a hook can still refuse after 0.10.0; the integer
error model and its overloaded codes; the 0-means-opaque alpha rule; and that none of it is thread
safe).
⚠ `cyrius doc` was NOT the answer and this is worth recording: it emits only the LAST LINE of each
doc comment, which for this tree's multi-paragraph headers is usually a fragment or a bare URL. The
reference is written by hand, its worked example is compiled and run, and its constant table is
machine-checked — but it is a SECOND copy of facts the headers own, so it will rot. ⇒ The headers
stay normative; where the two disagree the header is right and the reference is the bug.

**~~CI builds the host target only.~~ CLOSED in 0.9.1** — `.github/workflows/ci.yml` now carries a
"Cross-target link-check (aarch64 + AGNOS)" step that builds `programs/smoke.cyr` for both and fails
on the compiler's `raw syscall` portability diagnostic. VERIFIED locally across all four targets
(`x86_64`, `--aarch64`, `--agnos`, `--win`): zero raw-syscall warnings.
⚠ **What it does NOT do:** run anything. CI has neither an aarch64 nor an AGNOS host, so this is a
link-check — it proves sadish still *compiles* for AGNOS, which is what was missing, not that it
behaves there. Executing on AGNOS remains UNKNOWN from inside this repo.

### The AGNOS present path — and a correction

The `TODO` in `src/raster.cyr` reads *"a whole-canvas fast path routed through the AGNOS blit#39"*, and
the README carried it as a *"premultiplied `blit#39` fast path"*. ⛔ **That conflates two different
kernel calls.** VERIFIED against `agnos/docs/development/agnos-userland-abi.md`:

- **`blit`#39** — `blit(src, w, h, dstxy)`: *copies* a w×h block of 32bpp pixels from `src` to the
  framebuffer. It is a COPY, not a blend. ⚠ `src` must be **packed `w*4` per row**, which sadish
  surfaces are not required to be — `dh_surface_wrap` exists precisely to make wrapped ones, and
  0.6.0 made every primitive stride-correct. A fast path must therefore either require a packed
  surface or copy through one. The kernel gates `w * scale ≤ 8192` and rejects `scale > 16`, and the
  `defer` bit (a4 bit 40) lets a compositor accumulate windows and flip once with `present`#84.
- **`gpu_shader_op`#92 op 0x01** — premultiplied src-over on the shader cores. THIS is the
  premultiplied blend, and sadish has produced valid input for it since 0.7.0 (`src/premul.cyr`).

VERIFIED: the wrapper already exists in the vendored stdlib — `sys_blit(src, w, h, dstxy)` in
`lib/syscalls_x86_64_agnos.cyr`, `SYS_BLIT = 39`. sadish calls it nowhere.
⚠ The honest framing: this is a **presenter backend** (`src/present.cyr`, beside the Linux `/dev/fb0`
path), not a coverage-blit fast path — the `TODO` sits in `raster.cyr` next to `sd_canvas_blit_at`,
which composites coverage onto a surface and never touches a framebuffer. UNKNOWN from here: whether
aethersafha wants sadish presenting at all, or only producing premultiplied surfaces for the
compositor to blit. That answer decides whether this is a 1.0 item or not work at all.

### ~~`sd_present_open`'s device line~~ — GATED in 0.11.0

0.8.0 made everything after the `open()` reachable through `sd_present_open_fd` and recorded the rest
as needing a display. ⛔ **That was wrong, and 0.11.0 is the correction shipped.** `sd_present_open`
opens, probes geometry and allocates — it paints NOTHING. Only `sd_present_blit` writes pixels, and
only that call ever needed the "no test may touch the live display" rule.

`programs/present_geom_test.cyr` opens the real framebuffer, asserts the presenter's geometry against
**its own independent ioctl** on its own read-only descriptor, checks the fd is held and given back,
opens twice, and closes. ⭐ The independence is the point: a shared helper would agree with sadish by
construction and prove nothing. VERIFIED on this machine at **2560x1440, 32 bpp, pitch 10240**, with
nothing drawn. ⚠ It SKIPS, loudly, where there is no device or no permission, printing which branch
it took — both branches verified.

⇒ **What is left ungated is `sd_present_blit` alone**, the one call that genuinely needs a display,
and the one line between this suite and a test that draws on the user's screen.
⚠ UNKNOWN still: whether sadish should present at all on AGNOS — see the section above. A blit gate
would need a display sadish is allowed to scribble on, which is a different kind of infrastructure
(a virtual framebuffer, or a machine whose screen nobody minds), not a different test.

## Not sadish's

Consumer-side work that sadish cannot verify or perform is tracked in the consumer's own repo, not
here. The one open document in this repo is
[`proposals/2026-09-15-path-capacity-for-known-size-paths.md`](./proposals/2026-09-15-path-capacity-for-known-size-paths.md),
whose remaining item is an adoption decision that belongs to its filer.

## The cyrius pin: 6.6.4 → 6.6.6 — DONE in 0.9.1

**Pin is now `cyrius = "6.6.6"` (cyrius.cyml:8), `lib/` re-resolved against it.** The pre-flight
written here before the bump held item for item: **zero source changes were needed to compile.** Zero
`struct` declarations (so the new different-struct-copy error and the by-value >8 B deep-copy change
could not apply), no `async`/`operator` fns, no `ret2`/`rethi` pair returns, no SIMD-typed returns, no
top-level `{ }` blocks, no `: cstring` parameters, no duplicate global `var`s, no locally defined
`vec_*`. All 31 suites green on the first build.

⚠ **The two things the pre-flight did NOT predict**, both found by bumping and both fixed in 0.9.1:

1. **The 6.6.6 formatter re-indents continuation lines**, and `programs/paint_focal_test.cyr` had three
   that drifted. MEASURED: the 6.6.4 formatter rewrote that file to zero lines, 6.6.6 to three.
   ⛔ Worse, `cyrius fmt <file> --check` **exited 0 on it** while `cyrius audit`'s fmt stage failed it —
   so CI's format gate, which read that exit code, could not have caught the drift it existed to catch.
   The gate now formats in place and lets `git diff --exit-code` report. **Do not re-gate on `--check`.**
2. **The `O_TRUNC` note below was right about scope but pointed at the wrong call.** See the next
   section.

**The `lib/` re-resolve was the load-bearing step**, as predicted: the vendored fold was 12 libs behind
the store and every 0.9.0 build carried
`warning: ./lib/ shadows version-pinned …/lib — 12 bundled lib(s) differ`. `rm -rf lib && mkdir lib &&
cyrius deps` (what CI does) resolves **24 files** — the 9 declared `[deps] stdlib` leaves plus their
transitive peers — and clears the warning. ⚠ `cyrius update` instead vendors the **whole 111-file
snapshot**; same versions, but it is not what `.gitignore` and CI describe. Stay on `cyrius deps`.

## Raw syscall numbers: three of five closed, two deliberately open

The follow-on this file logged as "the hard-coded `2`/`1` are also wrong on aarch64" is **half closed**,
and the half that stayed open is the more useful record.

**Closed (0.9.1):** `SYS_WRITE` / `SYS_CLOSE` / `SYS_LSEEK` replace `1`/`3`/`8` at 11 sites in `src/`
(10 in `present.cyr`, 1 in `error.cyr`). The prompt was the compiler itself: `--aarch64` emitted
`src/present.cyr:376:36: raw syscall 8 is x86_64 lseek; on ELF-aarch64 that number is getxattr`.
⚠ Pre-existing — 6.6.4 warned too, only less precisely ("not one the aarch64 stdlib declares"); 6.6.6
naming `getxattr` is what made it worth closing. MEASURED: **all 32 built binaries sha256-identical**
before and after, so x86_64 codegen provably did not move.

**⛔ Deliberately still raw — and this is the correction to the note above.** The old entry framed the
open/write pair as one follow-on. They are not the same problem:

- **`open` (raw `2`, two sites)** — **aarch64 has no `open` syscall at all.** It has only `openat`, so
  the stdlib defines no `SYS_OPEN` for it. There is nothing to name.
- **`ioctl` (raw `16`, two sites)** — **AGNOS's ABI has no `ioctl`.** No `SYS_IOCTL` either.

MEASURED, not assumed: naming them fails the `--aarch64` and `--agnos` link-checks outright
(`undefined variable 'SYS_OPEN'` / `'SYS_IOCTL'`). ⇒ A wrong number on a path that cannot run on those
targets beats a build that does not link, because **every one of those four calls is the Linux
`/dev/fb0` sink**, which `src/present.cyr`'s header has always scoped as Linux-only. The reasoning is
written at the call site so a later reader does not "finish" the sweep and break the cross-build.
⇒ The real close for these two is the **AGNOS presenter backend** (kernel `blit#39`), not a rename —
and whether sadish should present at all is still the UNKNOWN recorded above.

⚠ **`programs/` keeps its raw numbers** and should: the suites are host-only and never cross-compiled,
so the collision cannot reach them. Only the shipped library is portable.
