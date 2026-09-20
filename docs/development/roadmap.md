# sadish roadmap

**Where we are:** 0.11.0. 34 RUN suites green; `fmt`, `lint`, `vet` and `distlib --check` clean;
`dist/sadish.cyr` in sync; toolchain pinned to **6.6.6**; the host build diagnostic-free and the
aarch64 + AGNOS cross-builds gated. agnos's `tests/gpu/refagree.cyr` holds the default fill
byte-identical across 200 random paths — green through every release since 0.6.0, and the standing
proof that a change did not move a pixel.

⛔ **THIS FILE IS WHAT IS LEFT, NOT WHAT WAS DONE.** Finished work is deleted from here once
[`../../CHANGELOG.md`](../../CHANGELOG.md) carries its measurements and a source header carries its
contract — keeping both is how a roadmap becomes an archive nobody reads. The one exception is
§"Decisions that still bind": rules a closed item left behind, which a later reader would otherwise
undo in good faith.

Each entry says how its claim was established: **VERIFIED** = read out of the tree while writing
this · **MEASURED** = a figure from the release that shipped it · **UNKNOWN** = cannot be settled
from inside this repo.

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

---

## The road to 1.0

⚠ **1.0 is a promise not to break the ABI**, and this project has broken it twice in four days —
0.9.0 on 2026-09-16, 0.10.0 on 2026-09-20, both inside its first eleven weeks. Both were right to
take, and both were only cheap because no version promised otherwise. Everything under "What 1.0
freezes" should be settled BEFORE the number is cut, because after it they are permanent.

### What 1.0 freezes

**The error model.** VERIFIED: `src/error.cyr` defines `SadishErr` — a 16 B record with a code and a
detail pointer — and **nothing in the library returns one, takes one, or stores one.** Its only
producer is `sadish_err_new`, its only consumers are the two accessors, and the only place producer
meets consumer is `programs/oom_test.cyr` group G, which builds two and discards them. The
`(ptr, err_out)` split `src/error.cyr` names as the record's purpose was never built: **zero**
functions take an `err_out`. Meanwhile EVERY status-returning entry point in the library answers
with a bare `SADISH_*` code, and always has.
⇒ Either the record is the contract or the codes are. Shipping 1.0 with both freezes a half-built
API permanently. ⚠ The cheap, honest answer is to **retire or demote the record** and document the
codes as the contract — but that is a decision, not a default. `docs/api.md` §2.4 currently tells
consumers to write against the codes, which pre-commits nothing.

**The overloaded codes, if the codes win.** VERIFIED, and worth settling in the same breath:
`SADISH_ERR_OOM` also means "a policy cap was hit", "a caller-supplied clip mask was null" and "a
0-area canvas"; a constructor's `0` collapses bad-argument, refused-allocation and — for
`sd_matrix_invert` — **genuinely singular**; `sd_path_flatten`'s `0` is *empty path* or *refused*,
told apart only by a process-wide flag. `SADISH_ERR_UNSUPPORTED` had **zero** return sites until
0.11.0. A caller cannot currently tell several of these apart, and 1.0 makes that permanent.

**What a hook can refuse — narrowed by 0.10.0.** VERIFIED: a fill, and a styled or dashed stroke, no
longer allocate on the `sd_alloc` seam at all, so a consumer's hook **cannot starve them**. Their
`SADISH_ERR_OOM` returns remain reachable only through the process scratch and the edge list, which
come from the global allocator and were never a hook's to refuse. ⚠ The ROUND stroker is the
exception and still allocates per piece.
⇒ The documented contract ("a refusal is a return code at every public entry point that can reach
one") is still true and now covers far fewer situations. Decide whether that asymmetry is the
shipped contract, or whether the round stroker should be brought in line. `src/alloc.cyr` states it
either way and `programs/stroke_oom_test.cyr` gates both halves.

### Capability gaps

**Arcs.** VERIFIED: the verb set is `SD_VERB_MOVETO / LINETO / QUADTO / CUBICTO / CLOSE`. SVG's `A`
and every rounded rectangle are hand-rolled by the caller into cubics today. Blast radius: a verb tag
plus a flattener; the verb/point stream pairing is already per-verb, and a new verb consumes its own
point count. ⚠ The alternative is to declare arcs a CONSUMER concern and document the cubic
approximation as the supported path — cheaper, and defensible for a core this size. **Decide which
before 1.0**, because adding a verb after it is an ABI change.

**Pattern paint's two follow-ons.** 0.11.0 shipped the kind; these were deliberately left.

1. **Bilinear filtering.** `sd_pattern_set_filter` already refuses anything but `SD_FILTER_NEAREST`
   with `SADISH_ERR_UNSUPPORTED`, and the filter is a FIELD — so this lands without an ABI break
   whenever it is wanted. It earns its cost only under a scaling or rotating matrix; at 1:1 nearest
   is exact and smoothing is a defect.
2. **A tile rect.** A pattern takes the WHOLE source surface. A consumer with a glyph atlas or a
   sprite sheet wants one sub-rect, and today must copy that rect into its own surface first.
   ⚠ This one IS ABI-shaped — a field on the paint record, or a second constructor — so it wants
   deciding before 1.0 even if it is not built.

**Blend modes.** VERIFIED: every compositor in the tree is src-over (`src/premul.cyr`,
`src/paint.cyr`, `src/raster.cyr`). No multiply/screen/darken. ⚠ UNKNOWN whether the consumers want
them: a dhancha/crab question, not a sadish one, and the answer decides whether this belongs in 1.0
at all.

### Infrastructure

**Nothing PROVES sadish runs on AGNOS.** VERIFIED: 0.9.1's cross-target step link-checks `--aarch64`
and `--agnos` and fails on the compiler's `raw syscall` diagnostic, which is what was missing — but
it **runs nothing**. CI has neither an aarch64 nor an AGNOS host. Compiling for AGNOS is proven;
behaving there is UNKNOWN from inside this repo and needs hardware or an emulator that is not ours
to add.

**`sd_present_blit` is the last ungated call.** 0.11.0 gated the device line —
`programs/present_geom_test.cyr` opens the real framebuffer and asserts its geometry against an
independent ioctl. What is left is the one call that genuinely writes pixels to a display. ⚠ A gate
would need a display sadish is allowed to scribble on — a virtual framebuffer, or a machine whose
screen nobody minds — which is infrastructure, not a different test.

### The AGNOS present path — and a correction

The `TODO` in `src/raster.cyr` reads *"a whole-canvas fast path routed through the AGNOS blit#39"*,
and the README once carried it as a *"premultiplied `blit#39` fast path"*. ⛔ **That conflates two
different kernel calls.** VERIFIED against `agnos/docs/development/agnos-userland-abi.md`:

- **`blit`#39** — `blit(src, w, h, dstxy)`: *copies* a w×h block of 32bpp pixels to the framebuffer.
  It is a COPY, not a blend. ⚠ `src` must be **packed `w*4` per row**, which sadish surfaces are not
  required to be — `dh_surface_wrap` exists precisely to make wrapped ones, and 0.6.0 made every
  primitive stride-correct. A fast path must therefore either require a packed surface or copy
  through one. The kernel gates `w * scale ≤ 8192`, rejects `scale > 16`, and its `defer` bit
  (a4 bit 40) lets a compositor accumulate windows and flip once with `present`#84.
- **`gpu_shader_op`#92 op 0x01** — premultiplied src-over on the shader cores. THIS is the
  premultiplied blend, and sadish has produced valid input for it since 0.7.0 (`src/premul.cyr`).

VERIFIED: the wrapper already exists in the vendored stdlib — `sys_blit(src, w, h, dstxy)` in
`lib/syscalls_x86_64_agnos.cyr`, `SYS_BLIT = 39`. sadish calls it nowhere.
⚠ The honest framing: this is a **presenter backend** (`src/present.cyr`, beside the Linux
`/dev/fb0` path), not a coverage-blit fast path — the `TODO` sits in `raster.cyr` next to
`sd_canvas_blit_at`, which composites coverage onto a surface and never touches a framebuffer.
⇒ **UNKNOWN, and it gates two other entries:** whether aethersafha wants sadish presenting at all,
or only producing premultiplied surfaces for the compositor to blit. That answer decides whether
this is a 1.0 item or not work at all — and it is also the real close for the raw `open` / `ioctl`
recorded below.

---

## Decisions that still bind

Rules a closed item left behind. Each cost something to learn, and each is the kind a later reader
would undo in good faith.

**Do not re-gate `fmt` on `cyrius fmt --check`.** MEASURED (0.9.1): it exits 0 on files the 6.6.6
formatter would still rewrite, while `cyrius audit`'s fmt stage fails them. CI formats in place and
lets `git diff --exit-code` report.

**Resolve `lib/` with `cyrius deps`, not `cyrius update`.** `deps` vendors the 9 declared stdlib
leaves plus their transitive peers — 24 files, which is what `.gitignore` and CI describe. `update`
vendors the whole 111-file snapshot: same versions, wrong shape.

**`open` and `ioctl` stay raw in `src/present.cyr`; do NOT "finish" the syscall sweep.** 0.9.1
replaced `1`/`3`/`8` with `SYS_WRITE` / `SYS_CLOSE` / `SYS_LSEEK` at 11 sites. The other two have no
portable spelling: **aarch64 has no `open`** (only `openat`, so no `SYS_OPEN`) and **AGNOS has no
`ioctl`**. MEASURED: naming them fails the `--aarch64` and `--agnos` link-checks outright. All four
remaining raw calls belong to the Linux `/dev/fb0` sink, which is Linux-only by design; the real
close is the AGNOS presenter backend, not a rename. The reasoning is written at the call site.
⚠ `programs/` keeps its raw numbers and should — the suites are host-only and never cross-compiled.

**`docs/api.md` is a SECOND copy; the source headers are normative.** Where the two disagree the
header is right and the reference is the bug. ⚠ `cyrius doc` cannot generate it: it emits only the
LAST LINE of each doc comment, which for this tree's multi-paragraph headers is usually a fragment.

**`sadish_version()` is derived from `VERSION` by CI.** It returned 900 through 0.9.1 AND 0.10.0 —
across an ABI break — because the gate asserted the same stale literal. Bump both, or CI stops you.

**Every `docs/development/` path must resolve, and must never be elided.** `src/*.cyr` comments are
copied verbatim into `dist/sadish.cyr`, so a stale path ships to every consumer. Cross-repo filings
carry their owning repo (`rekha/…`, `dhancha/…`, `agnos/…`). The sweep is in
[`issues/README.md`](./issues/README.md); it is clean as of 0.11.0.

---

## Not sadish's

Consumer-side work that sadish cannot verify or perform is tracked in the consumer's own repo, not
here.

⭐ **There is no open filing in this repo.** `docs/development/issues/` and `proposals/` are both
fully archived as of 0.11.0 — the last one,
[`proposals/archived/2026-09-15-path-capacity-for-known-size-paths.md`](./proposals/archived/2026-09-15-path-capacity-for-known-size-paths.md),
closed when its remaining item landed in **rekha 0.4.3** (the adoption this repo could not witness:
`rekha_outline_to_sdpath` now opens each glyph at `sd_path_new_cap(v, p)` and `face_test` asserts the
estimate is exact across all 2,620 LiberationSans glyphs). ⚠ Archived is not deleted — a filing is
the record of what was measured, and the measurement outlives the bug. What remains open is in this
file, not in a filing.
