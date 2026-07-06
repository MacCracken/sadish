# sadish — 2D prior-art survey (ecosystem games)

Mined 2026-07-04 from the Cyrius games that already hand-roll 2D drawing, to
decide what sadish **adopts / extracts up** vs **writes fresh**. Sources:
`cyrius-bb`, `cyrius-polyomino`, `encom-hits` (and `cyrius-doom`, which shares
the same present layer). This is the "redesign don't reinvent" pass: the prior
art is *inside* the ecosystem, not just external.

## Verdict

The games give sadish a **floor** (opaque axis-aligned fills + hairline
Bresenham lines), a mature **Linux `/dev/fb0` present backend**, and a **line
skeleton** — but **zero** for sadish's hard parts. Those are net-new, and this
validates the scaffold's TODOs (`path.sd_path_flatten` + `raster` coverage-fill
are exactly the net-new work).

## EXTRACT UP — lift into sadish; games become thin consumers (kashi/akshara pattern)

1. **Linux `/dev/fb0` present backend** — the strongest artifact, and it is
   **duplicated across cyrius-bb, cyrius-polyomino, encom-hits, AND cyrius-doom**
   (encom's comment cites the shared doom/polyomino bug fix). encom-hits'
   `engine.cyr` is the most mature: probe `FBIOGET_VSCREENINFO`/`FSCREENINFO` →
   largest integer scale that fits → center-letterbox → honor `line_length`
   (pitch) → 32bpp verbatim + 16bpp RGB565 pack paths → PPM fallback for
   headless/CI. Lift into sadish's **output/backend module** as the Linux host
   sink, beside the future agnos `blit`#39 sink. Extracting it **de-dupes 4
   repos** — the biggest single consolidation win here.
   (`encom-hits/src/engine.cyr:111-259`)
2. **Integer Bresenham line + rect + pixel/clear** — `encom-hits/src/draw.cyr`
   (`draw_line`/`hline`/`vline`/`rect`/`filled_rect`/`pixel`/`clear`). Clean,
   dependency-free. sadish's hairline + flattened-curve segment path. Adopt the
   *structure*; add an AA/coverage variant alongside (Bresenham is opaque 1px
   only). (`encom-hits/src/draw.cyr:7-141`)
3. **Arc/circle flatten template** — `encom-hits/src/mcpcone.cyr`: a sin/cos
   ×100 fixed-point table → emit N line segments (polar→cartesian, integer
   ÷100). A fixed-point integer fast-path for sadish's arc/circle flattening.
   (`mcpcone.cyr:31-72, 184-188`)
4. **`fill_rect` = degenerate axis-aligned scanline fill** —
   `cyrius-bb`/`cyrius-polyomino` `fb_fill_rect`. Conceptual boundary: sadish's
   general scanline fill (edge crossings → spans → coverage) *subsumes* it.
   **Generalize, don't copy** (span-run fills, not per-pixel `fb_set`).
   (`cyrius-polyomino/src/framebuf.cyr:57-79`)
5. **span-clamp clipping** — hline/vline endpoint clamp is the right idiom for
   sadish's scanline span emitter (clamp `[x0,x1]` to the clip rect per row).

## ADOPT — idioms / conventions

- **BGRX / `0x00RRGGBB` packed color** matching LE 32bpp fb — keeps the
  coverage→pixel blit a verbatim copy. Converge on it (all three games already do).
- **Offscreen-surface-first + PPM debug dump** — validated across all three;
  keep sadish's coverage/pixel buffer independent of the sink and ship a PPM
  dumper for pixel-assert CI (headless-testable).
- **⚠ `asr()` signed arithmetic-right-shift** (`cyrius-bb/src/fixed.cyr:22`) —
  **LOAD-BEARING.** Cyrius `>>` is **logical** (zero-fill), so any signed
  subpixel / coverage / fixed-point math in sadish's flattener + coverage
  accumulator MUST use a hand-rolled arithmetic-right-shift, or negatives
  silently corrupt. The games mostly dodge this by staying integer; sadish's
  subpixel coverage cannot.
- **`{w,h,px}` heap struct + `off=(y*w+x)*bpp` + top-left / y-down** — the
  ecosystem coordinate/layout convention.

## WRITE FRESH — no prior art anywhere (the hard 20%)

- Anti-aliased **coverage** accumulation (all games are opaque-only).
- **Scanline fill** with edge crossings + nonzero / even-odd **winding**.
- **Bézier flattening** (quad/cubic → polyline; adaptive de Casteljau) — zero
  curve code exists in any game.
- Affine **`SdMatrix`** (rotate / scale / skew) — encom's "rotation" is an
  index-offset into a 16-entry table, not a matrix.
- General **line/polygon clipping** (Cohen-Sutherland / Liang-Barsky).
- **Gradients** + **source-over alpha** blend (only opaque + one additive-glow
  post-pass exist).

## Handoff to rekha (fonts, above sadish)

The bitmap-glyph **dispatch/layout glue** (bb/polyomino/encom `hud.cyr` /
`main.cyr`: 3×5 packed-bit font, ASCII routing, advance-width, base-10 digit
extraction) belongs to **rekha** as a fallback bitmap-font path — the
rasterization drops to sadish; rekha replaces the bitmap with vector outlines →
sadish fill.

## Suggested v0.2 sequence

1. Lift encom `draw.cyr` (line/rect/pixel) + `engine.cyr` (present backend) →
   sadish's primitive floor + Linux sink; make **encom-hits the first consumer**.
2. Copy cyrius-bb's `asr()` fixed-point discipline into `geom`'s fixed-point.
3. Build the write-fresh list on top: scanline coverage fill → Bézier flatten
   (wire `sd_path_flatten`) → `SdMatrix` rotate → clipping → gradients/alpha.

---

*Source files — encom-hits: `src/{draw,engine,mcpcone,types}.cyr`,
`src/main.cyr:88-172` (3×5 font, NOT for sadish). cyrius-bb:
`src/{framebuf,present,fixed,hud}.cyr`. cyrius-polyomino:
`src/{framebuf,present,hud}.cyr`.*
