# sadish roadmap

**Where we are:** 0.9.0, tagged. 31 RUN suites green, `fmt`/`lint`/`vet` clean, `dist/sadish.cyr` in
sync. agnos's `tests/gpu/refagree.cyr` holds the default fill byte-identical across 200 random paths —
it has stayed green through every release since 0.6.0 and is the standing proof that a change did not
move a pixel.

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

## The road to 1.0

⚠ **1.0 is a promise not to break the ABI**, and 0.9.0 broke it three weeks into the project's life.
Everything under "What 1.0 freezes" should be settled BEFORE the number is cut, because after it they
are permanent.

### Capability gaps

**Image / pattern paint.** VERIFIED: `src/paint.cyr` defines `SD_GRADIENT_LINEAR`, `_RADIAL`,
`_FOCAL` and the three spreads — there is no paint that samples a source surface. A consumer can fill
with a colour or a gradient and nothing else, so a canvas/SVG layer cannot draw a bitmap, a repeating
pattern, or a cached tile. Blast radius: a new paint kind beside `SdGradient` plus a sampler in the
blit loops; the coverage engine is untouched. ⚠ Needs a decision on filtering (nearest vs bilinear)
and on what a pattern does outside its source rect — the same spread question gradients already
answer, so the shape exists to copy.

**Arcs.** VERIFIED: the verb set is `SD_VERB_MOVETO / LINETO / QUADTO / CUBICTO / CLOSE`. SVG's `A`
and every rounded rectangle are hand-rolled by the caller into cubics today. Blast radius: a verb tag
plus a flattener; the verb/point stream pairing is already per-verb, and a new verb consumes its own
point count. ⚠ The alternative is to declare arcs a CONSUMER concern and document the cubic
approximation as the supported path — cheaper, and defensible for a core this size.

**`sd_path_transform`.** VERIFIED: `SdMatrix` and `sd_matrix_apply` exist, but nothing applies a matrix
to a path — `grep 'fn sd_path_transform' src/` is empty. Every consumer that scales or rotates a path
writes the loop itself. Blast radius: one function over the inline points array, which 0.9.0 made
trivial (two i64 per slot, no records to rebuild).

**Path bounds.** VERIFIED: no bounds query exists. `SdPath`'s reserved word was annotated *"future
flags: subpath-open, bounds-cached"* and 0.7.2 spent it on the point capacity instead, so a cached
bound now needs a new field or a recompute. Culling, layout and damage tracking all want it.

**Blend modes.** VERIFIED: every compositor in the tree is src-over (`src/premul.cyr`, `src/paint.cyr`,
`src/raster.cyr`). No multiply/screen/darken. ⚠ UNKNOWN whether the consumers want them: that is a
dhancha/crab question, not a sadish one, and the answer decides whether this belongs in 1.0 at all.

### What 1.0 freezes

**The `SdPolyline` / `SdPath` split.** VERIFIED: 0.9.0 put `SdPath`'s points inline, but `SdPolyline`
still holds *"an array of i64 `SdPoint` ptrs"* (`src/path.cyr`, the record comment), and its consumers
read them through `sd_polyline_points` + `sd_point_x`. So the library now stores points two different
ways and hands consumers both. Making them consistent is the same class of break 0.9.0 just took —
cheaper now than after 1.0, and the accessors that made 0.9.0 survivable (`sd_path_point_x` / `_y`)
give the pattern to copy.

**The error model.** VERIFIED: `src/error.cyr` defines `SadishErr` with codes and a detail pointer, and
almost nothing constructs one — the library returns bare `SADISH_*` codes and 0. Either the record is
the contract or the codes are; shipping 1.0 with both leaves a half-built API permanently.

### Infrastructure

**No API reference.** VERIFIED: `docs/` holds `development/{issues,proposals,prior-art.md}` and now
this file. Every contract lives in a source header, so a consumer learns the library by reading
`dist/sadish.cyr` — 7,866 lines. The headers are good; they are just not reachable as documentation.

**CI builds the host target only.** VERIFIED: `.github/workflows/ci.yml` builds `programs/smoke.cyr`
and each suite with no `--agnos` pass. AGNOS is the point of the stack and nothing proves sadish still
compiles for it.

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

### `sd_present_open`'s device line — gateable, and wrongly written off

MEASURED (0.8.0): pointing it at `/dev/fb1`, or replacing its body with `return 0;`, leaves all 31
suites green, so the device path itself is unpinned. 0.8.0 made everything after the `open()` reachable
through `sd_present_open_fd` and recorded the rest as needing a display.

⛔ **That was wrong, and this is the correction.** `sd_present_open` opens, probes geometry and
allocates — it paints NOTHING. Only `sd_present_blit` writes pixels, and only that call has ever needed
the "no test may touch the live display" rule. VERIFIED on this machine: a probe calling
`sd_present_open()` then `sd_present_close()` opened the real framebuffer and read back
**2560x1440, 32 bpp, pitch 10240**, with nothing drawn.

⇒ The gate is a suite that calls `sd_present_open()`, asserts the presenter's fields against the
device's own geometry, and closes — never blitting. ⚠ It must SKIP cleanly where there is no
framebuffer or no permission (`/dev/fb0` is `root:video`; CI has no device at all), and say which
branch it took, so a skip is never mistaken for a pass. That leaves only the blit itself ungated,
which is the call that genuinely needs a display.

## Not sadish's

Consumer-side work that sadish cannot verify or perform is tracked in the consumer's own repo, not
here. The one open document in this repo is
[`proposals/2026-09-15-path-capacity-for-known-size-paths.md`](./proposals/2026-09-15-path-capacity-for-known-size-paths.md),
whose remaining item is an adoption decision that belongs to its filer.
