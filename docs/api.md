# sadish API reference

**v0.11.0** — 129 public functions. Written by hand from the source headers, which remain the
normative text: where this file and a `src/*.cyr` header disagree, the header is right and this file
is a bug.

This exists because a consumer used to learn sadish by reading `dist/sadish.cyr` — 8,000+ lines.
It is a **reference**, not a tutorial: it says what each call answers and what it costs, and it
front-loads the six cross-cutting rules that most consumer bugs come from.

---

## 1. The shape of the library

sadish turns resolution-independent geometry into **coverage**, and coverage into **pixels**.

```
  SdPath ──flatten──▶ SdPolyline ──┐
                                   ├──▶ SdCanvas (coverage, 1 byte a pixel)
  fill / stroke / clip ────────────┘              │
                                                  │  blit + a paint
                                                  ▼
                                            SdSurface (BGRA pixels)
                                                  │
                                                  ▼
                                     PPM file, or SdPresenter → /dev/fb0
```

Five records, all opaque pointers you get from a constructor and never free (see §2.2):

| record | made by | holds |
|---|---|---|
| `SdSurface` | `sd_surface_new(w, h)` | BGRA pixels + a stride |
| `SdCanvas` | `sd_canvas_new(w, h)` | one coverage byte a pixel, an AA mode, a clip stack |
| `SdPath` | `sd_path_new()` / `_cap()` | a verb stream + an inline point stream |
| `SdPolyline` | `sd_path_flatten(path, tol)` | flattened points + a verdict |
| `SdGradient` | `sd_gradient_*()` / `sd_pattern_new()` | a paint — read it as **SdPaint** |
| `SdMatrix` | `sd_matrix_*()` | a 2×3 affine |
| `SdPresenter` | `sd_present_open*()` | a framebuffer fd + probed geometry |

⚠ `SdPoint` still exists (`sd_point_new`) and is taken/returned by `sd_matrix_apply`, but since
0.10.0 **no store in sadish is an SdPoint** — both `SdPath` and `SdPolyline` hold coordinates inline.
It is an argument type now, nothing more.

---

## 2. The six rules

### 2.1 Coordinates are 16.16 fixed point — and `>>` is not a shift you want

Geometry is `i64` **16.16**: `SD_ONE = 65536` is one pixel. There is no float anywhere.

⛔ **Cyrius's `>>` is LOGICAL (zero-fill).** A plain `>>` on a negative coordinate destroys it. Use
`sd_asr(x, n)`. This is not a style note — it is the single most common way to write a bug against
this library, and sadish itself shipped one in 0.11.0's first draft (pattern sampling read texel
`-3` as a huge positive index; `programs/pattern_test.cyr` group D is the gate).

⚠ **Precision bound.** Every coordinate must satisfy |v| < 16,383 px (16.16 magnitude < 2^30) so
that products of two such values stay inside i64. Paints state it again because they multiply pairs.

⚠ **Pixel centres.** Paints sample at the pixel **centre** `(x + 0.5, y + 0.5)`. The legacy
`sd_canvas_blit_gradient` samples the top-left corner instead; the two are deliberately not
interchangeable.

### 2.2 There is no `free()` — there is a seam

`sd_alloc(n)` is the one place sadish takes memory. By default it is the stdlib bump allocator,
which has **no free**. A consumer installs its own:

```
var prev = sd_alloc_set(&my_arena_alloc);
... build paths, draw ...
sd_alloc_set(prev);
```

⛔ **A pointer that outlives the hook's backing store is dangling.** If the hook is a per-frame
arena that gets reset, nothing sadish returned under it may be kept across the reset — not a path,
not a canvas, not a surface. That is the consumer's lifetime call.

⚠ **The fill/stroke scratch is deliberately NOT on the seam** — it is taken once per process from
the global allocator, so a reset arena cannot leave it dangling. A hook therefore never sees it.

### 2.3 What a refusal costs — and what a hook can still refuse

A hook **may return 0**. Every `sd_alloc` result in `src/` is checked.

- **Constructors answer 0.** `sd_surface_new`, `sd_canvas_new`, `sd_path_new`/`_cap`,
  `sd_point_new`, every `sd_matrix_*`, the gradient and pattern constructors, `sd_path_flatten`,
  the presenter. ⇒ **Test a constructor's result before building on it. That is the whole of the
  caller's duty** — passing a documented 0 back into sadish is the one thing that still faults.
- **Status calls answer `SADISH_ERR_OOM`.** The path builders, the clip pushes,
  `sd_surface_write_ppm`, the gradient setters, the fills and all three stroke entries.
- ⚠ **A fill or stroke is not all-or-nothing**: coverage written before a refusal stays on the
  canvas. Clear and retry, or draw into a scratch canvas. The path builders and clip pushes *are*
  all-or-nothing.

⭐ **Since 0.10.0 a hook can refuse far less than it could.** A flattened point costs no allocation,
so **a fill, and a styled or dashed stroke, ask the seam for nothing at all** and cannot be starved
through it. Their `SADISH_ERR_OOM` returns remain reachable only via the process scratch and the
edge list, which come from the global allocator. ⚠ The **round** stroker is the exception — it
still builds a path per segment and per vertex on the seam, and starves exactly as before.

### 2.4 Errors are integer codes

```
SADISH_OK = 0   SADISH_ERR_OOM = 1   SADISH_ERR_EMPTY_PATH = 2
SADISH_ERR_BOUNDS = 3   SADISH_ERR_UNSUPPORTED = 4   SADISH_ERR_OTHER = 99
```

⚠ **The `SadishErr` record (`sadish_err_new`, `sadish_err_code`, `sadish_err_detail`) is not part of
any call's contract.** No function in the library returns one, takes one, or stores one. It is a
stub from 0.1.0 whose promised `(ptr, err_out)` split was never built, and the roadmap lists
settling it as a **1.0 blocker**. Write against the codes.

⚠ Codes that are **overloaded**, stated so a caller is not surprised:

- `SADISH_ERR_OOM` also means "a policy cap was hit" (`sd_path_new_cap` past `SD_PATH_CAP_MAX`), "a
  caller-supplied clip mask was null", and "a 0-area canvas".
- A constructor's `0` collapses every cause: bad argument, refused allocation, and — for
  `sd_matrix_invert` — a genuinely **singular** matrix. For `sd_path_flatten`, `0` is *empty path*
  or *refused*, told apart only by `sd_flatten_truncated()`.
- `SADISH_OK` from `draw.cyr`'s primitives means nothing at all: they always return 0 and clip
  off-surface writes silently.

### 2.5 Colour: 0xAARRGGBB, and alpha 0 means opaque

`sd_rgb(r, g, b)` returns `0x00RRGGBB` — byte 3 is **0, and 0 reads as OPAQUE** (`sd_alpha_of`).
That legacy rule exists because treating 0 as transparent would have made every pre-existing drawing
vanish. The cost: **fully transparent is not expressible as a colour**; a caller that wants to draw
nothing does not draw.

⚠ The mapping is resolved at **store** time — `sd_plot` and every `draw.cyr` primitive write
`sd_alpha_of(colour)` — so a surface drawn with `sd_rgb` holds **255** in memory. That is why
sampling raw bytes (`sd_surface_rgba_at`, and pattern paint) is safe: a byte-3 of 0 in a surface is
a pixel nothing wrote, or one written transparent on purpose, and both mean transparent.

⚠ **Two alpha worlds.** The straight blits force destination alpha 255 — they cannot produce a
transparent result. To composite onto a transparent window, use the **premultiplied** entries in
`premul.cyr`. Feed `gpu_shader_op#92` only premultiplied data; passing straight alpha renders washed
out with no diagnostic anywhere in the stack.

### 2.6 It is not thread safe

The allocation hook, the fill/stroke scratch, the flatten budget and the flatten verdict flags are
**process-wide globals**. sadish has never had a lock. Draw from one thread.

---

## 3. Reference by module

Signatures are `name(args)`; every function returns `i64`. "→" gives the meaning of the return.

### 3.1 `alloc.cyr` — the seam

| function | → |
|---|---|
| `sd_alloc(n)` | n bytes through the hook, or the global allocator; **0 on failure** |
| `sd_alloc_set(fp)` | installs hook `fp` (0 restores the global one); returns the **previous** hook |
| `sd_alloc_get()` | the current hook, 0 = global |

### 3.2 `error.cyr` — codes

| function | → |
|---|---|
| `sadish_err_name(code)` | a cstring, never null; unknown codes give `"unknown error"` |
| `sadish_err_print_name(code)` | writes the name to stderr; returns 0 |
| `sadish_err_new(code, detail)` / `sadish_err(code)` | a 16 B record, or 0 |
| `sadish_err_code(e)` / `sadish_err_detail(e)` | the field; **`sadish_err_code(0)` is `SADISH_ERR_OOM`**, never OK |

⚠ See §2.4: none of this is in any call's contract.

### 3.3 `surface.cyr` — pixels and colour

| function | → |
|---|---|
| `sd_surface_new(w, h)` | an `SdSurface`, or 0 |
| `sd_surface_width/height/pixels/stride(s)` | the field |
| `sd_surface_pixel_at(s, x, y)` | `0x00RRGGBB`, **alpha dropped**; 0 out of bounds |
| `sd_rgb(r, g, b)` / `sd_rgba(r, g, b, a)` | a packed colour (`a` is 1..255; 0 reads opaque) |
| `sd_color_r/g/b(c)`, `sd_color_a(c)` | channels; `sd_color_a` is the **raw** byte |
| `sd_alpha_of(c)` | the alpha it will draw with — **0 → 255** |
| `sd_premul(c)` | each channel scaled by its own alpha |

⚠ **Rows are addressed by the STRIDE, not by the width.** A wrapped surface (a window onto a bigger
buffer) has stride > w*4, and the bytes between one row's end and the next are not its pixels.

### 3.4 `draw.cyr` — direct opaque primitives

`sd_plot(s, x, y, color)` · `sd_line(s, x0, y0, x1, y1, color)` · `sd_hline(s, x0, x1, y, color)` ·
`sd_vline(s, x, y0, y1, color)` · `sd_rect(s, x, y, rw, rh, color)` ·
`sd_fill_rect_blend(s, x, y, rw, rh, color, a)` · `sd_blend_hline(s, x0, x1, y, color, a)` ·
`sd_clear(s, color)`

⚠ **Integer pixel coordinates, not 16.16** — these are the "easy 80%" primitives, not the
rasterizer. They always return 0, clip silently, and never allocate.

### 3.5 `geom.cyr` — fixed point and affines

| function | → |
|---|---|
| `sd_asr(x, n)` | **sign-preserving** right shift — see §2.1 |
| `sd_abs(v)`, `sd_isqrt(n)` | absolute value, integer square root |
| `sd_point_new(x, y)`, `sd_point_x/y(p)` | a 16 B `SdPoint`, or 0 |
| `sd_matrix_identity/translate/scale/rotate/skew(...)` | an `SdMatrix`, or 0 |
| `sd_matrix_mul(m, n)` | m∘n, or 0 |
| `sd_matrix_invert(m)` | the inverse, **or 0 for a singular matrix** (indistinguishable from OOM) |
| `sd_matrix_apply(m, pt)` | a **new** `SdPoint` — one allocation per point |
| `sd_cos(theta)`, `sd_sin(theta)` | 16.16, via CORDIC; `theta` in 16.16 radians |

⭐ To transform a whole path, use `sd_path_transform` (§3.6) — it allocates nothing, where a
`sd_matrix_apply` loop costs one record per point.

### 3.6 `path.cyr` — geometry

**Build**

| function | → |
|---|---|
| `sd_path_new()` | a path at default capacity, or 0 |
| `sd_path_new_cap(n_verbs, n_points)` | a path at **exactly** those capacities, or 0 |
| `sd_path_moveto/lineto(path, x, y)` | `SADISH_OK` / `SADISH_ERR_OOM` |
| `sd_path_quadto(path, cx, cy, x, y)` | ” (consumes 2 points) |
| `sd_path_cubicto(path, c1x, c1y, c2x, c2y, x, y)` | ” (3 points) |
| `sd_path_close(path)` | ” (0 points) |

**Read** — ⛔ *always* through these, never by indexing the record:

`sd_path_verb_count(path)` · `sd_path_point_count(path)` · `sd_path_verb_cap(path)` ·
`sd_path_point_cap(path)` · `sd_path_point_x(path, i)` · `sd_path_point_y(path, i)` ·
`sd_path_verb_at(path, i)` → one of `SD_VERB_MOVETO/LINETO/QUADTO/CUBICTO/CLOSE`

⚠ No bounds checks — bound the loop on the count.

**Transform and measure** (0.10.0)

| function | → |
|---|---|
| `sd_path_transform(path, m)` | affines every point **IN PLACE**, allocating nothing. `SADISH_OK` / `SADISH_ERR_BOUNDS` |
| `sd_path_bounds(path, out)` | 4 × 16.16 into a caller's 32 B buffer (min x, min y, max x, max y). `SADISH_ERR_EMPTY_PATH` **leaves `out` unwritten** |

⚠ `sd_path_transform` destroys the original geometry, and transforms the **control** points — exact
for affine, so curves are not re-approximated. ⚠ `sd_path_bounds` is the **control-point hull**: a
curve's box contains the drawn shape and may exceed it. Conservative is what culling wants; for the
tight box, flatten first and bound the polyline.

**Flatten**

| function | → |
|---|---|
| `sd_path_flatten(path, tol)` | an `SdPolyline`, or 0 (**empty path or refused** — see §2.4) |
| `sd_polyline_count(pl)` | points |
| `sd_polyline_point_x/y(pl, i)` | the coordinate, 16.16 |
| `sd_polyline_points(pl)` | the raw block — ⛔ slots are **16 B coordinate pairs** since 0.10.0, not pointers |
| `sd_polyline_verdict(pl)` | `SD_POLYLINE_TRUNCATED` \| `SD_POLYLINE_DEGRADED`, 0 = whole |
| `sd_polyline_truncated/degraded(pl)` | 1/0, for **this** result |

⚠ **Truncated** = points were lost, the contour is open where it stops. **Degraded** = every point
is present but some curve was cut to its chord by the work budget. Ask the *record*, not the globals.

**The work budget** — `sd_flatten_budget_set(n)` / `_get()`, `sd_flatten_op_begin()` / `_op_end()`,
`sd_flatten_degraded()` / `_truncated()` (process-wide and **sticky**; the record is the per-result
answer). `sd_quad_flat` / `sd_cubic_flat` expose the flatness predicates.

### 3.7 `raster.cyr` — coverage

| function | → |
|---|---|
| `sd_canvas_new(w, h)` | an `SdCanvas`, or 0 |
| `sd_canvas_width/height/coverage/stride/aa(cv)` | the field |
| `sd_canvas_set_aa(cv, mode)` | `SD_AA_SUBSCANLINE` (default) or `SD_AA_AREA` (exact 2-axis) |
| `sd_canvas_clear(cv)` | `SADISH_OK` — ⚠ **no null check**, `cv == 0` faults |
| `sd_canvas_fill_path(cv, path, rule)` | rule = `SD_FILL_EVEN_ODD` / `SD_FILL_NONZERO` |
| `sd_canvas_fill_union(cv, path, rule)` | fill, ORed into existing coverage |
| `sd_canvas_blit_at(cv, surface, color, dx, dy)` | composite coverage as a solid colour |
| `sd_canvas_blit(cv, surface, color)` | ” at the origin |
| `sd_canvas_clip_push_rect(cv, x0, y0, x1, y1)` | ⚠ **PIXEL** coordinates, not 16.16 |
| `sd_canvas_clip_push_path(cv, path, rule)` | intersect a filled path into the clip |
| `sd_canvas_clip_pop(cv)` | pop one level |
| `sd_grow_limit_set(n)` / `_get()` | the growable-store ceiling in **bytes** |
| `sadish_version()` | `major*10000 + minor*100 + patch` — 0.11.0 → `1100` |

⚠ `sd_canvas_blit_gradient(cv, surface, x0, y0, c0, x1, y1, c1)` is the **legacy** 2-stop gradient:
whole-pixel endpoints, corner sampling. Kept working, superseded by §3.9.

### 3.8 `stroke.cyr` / `dash.cyr`

| function | → |
|---|---|
| `sd_canvas_stroke_path(cv, path, width)` | the **round** stroker (discs + segment rects) |
| `sd_canvas_stroke_path_ex(cv, path, width, cap, join, miter_limit)` | caps `SD_CAP_BUTT/ROUND/SQUARE`, joins `SD_JOIN_MITER/ROUND/BEVEL` |
| `sd_canvas_stroke_path_dash(cv, path, width, cap, join, miter_limit, pattern, n, offset)` | `pattern` = n on/off lengths, 16.16 |

⚠ `width` and the dash lengths are 16.16. A `width <= 0` is `SADISH_OK` and paints nothing.
⚠ A null/empty dash pattern silently falls back to a **solid** stroke and reports success.
⭐ The styled and dashed walks allocate nothing on the seam; the round stroker allocates per piece.

### 3.9 `paint.cyr` — gradients and patterns

A **paint** is a colour for every point of canvas space. Build it once, then blit through it.

| function | → |
|---|---|
| `sd_gradient_linear(x0, y0, x1, y1)` | a paint, or 0 |
| `sd_gradient_radial(cx, cy, r)` | ” |
| `sd_gradient_radial_focal(cx, cy, r, fx, fy)` | ” (focal point clamped inside r) |
| `sd_pattern_new(src)` | **0.11.0** — a paint that samples surface `src`, or 0 |
| `sd_gradient_add_stop(g, offset, color)` | offset 16.16 in [0, 1]; stable sort, any order |
| `sd_gradient_set_spread(g, spread)` | `SD_SPREAD_PAD` (default) / `REPEAT` / `REFLECT` / `NONE` |
| `sd_gradient_set_matrix(g, m)` | `m` maps **paint space → canvas space** |
| `sd_gradient_stop_count(g)`, `sd_gradient_color_at(g, t)` | inspection |
| `sd_paint_is_pattern(p)`, `sd_pattern_source(p)` | 1/0, the borrowed surface |
| `sd_pattern_filter(p)`, `sd_pattern_set_filter(p, f)` | `SD_FILTER_NEAREST` only; anything else is `SADISH_ERR_UNSUPPORTED` |
| `sd_canvas_blit_paint_at(cv, surface, g, dx, dy)` | composite coverage through the paint |
| `sd_canvas_blit_paint(cv, surface, g)` | ” at the origin |

**Patterns** (0.11.0): one source texel per canvas pixel, nearest-neighbour. Unmapped, texel (0,0)
sits on canvas pixel (0,0) — a 1:1 tile blit. ⚠ The source is **borrowed, not copied**: it must
outlive the pattern, and mutating it changes what the pattern paints.

⚠ **`SD_SPREAD_NONE` is a pattern's alone** — "paint nothing outside the source rect". It **skips**
those pixels rather than writing them transparent, which matters because the straight blit forces
destination alpha 255. A gradient asking for it gets `SADISH_ERR_UNSUPPORTED`: a gradient's `t` is
defined everywhere, so there is no rect to be outside of.

⚠ Spreads use **floor** semantics on negatives, patterns and gradients alike: under `REPEAT` texel
−1 is texel n−1, and gradient `t = −0.25` reads 0.75.

⚠ A paint with **zero stops paints nothing** and returns `SADISH_OK` (SVG's `none`); one stop is
solid. A blocked (singular) matrix likewise paints nothing at blit time, though `sd_gradient_set_matrix`
returned `SADISH_ERR_BOUNDS` when it was set.

### 3.10 `premul.cyr` — premultiplied output

| function | → |
|---|---|
| `sd_surface_rgba_at(s, x, y)` | `0xAARRGGBB`, **all four bytes raw**; 0 out of bounds |
| `sd_clear_premul(s, color, a)` / `sd_fill_rect_premul(s, x, y, w, h, color, a)` | overwrite premultiplied at alpha `a` |
| `sd_canvas_blit_premul_at(cv, surface, color, a, dx, dy)` / `_premul(...)` | solid, premultiplied |
| `sd_canvas_blit_paint_premul_at(cv, surface, g, dx, dy)` / `_premul(...)` | any paint, premultiplied |

⭐ These are what produce valid input for AGNOS's `gpu_shader_op#92` op 0x01, and the only way to get
a genuinely **transparent** result — the straight blits force destination alpha 255.

### 3.11 `present.cyr` — getting a surface onto a screen

| function | → |
|---|---|
| `sd_surface_write_ppm(s, path)` | binary P6 PPM; the headless/CI/debug sink |
| `sd_present_open()` | open `/dev/fb0`, probe geometry → an `SdPresenter`, or 0 |
| `sd_present_open_path(path)` / `_fd(fd)` | the same, by name / onto a descriptor you hold |
| `sd_present_blit(p, s)` | ⛔ **writes to the live display** |
| `sd_present_close(p)` | closes the fd; `SADISH_OK` always, 0 is a documented no-op |

⚠ The fb0 path is **Linux-specific**. The AGNOS sink (kernel `blit#39`) is a separate backend that
does not exist yet. ⚠ `sd_present_open` opens, probes and allocates but **paints nothing** — only
`sd_present_blit` touches the display.

---

## 4. Constants

**Fixed point** `SD_ONE` = 65536 · `SD_SHIFT` = 16
**Verbs** `SD_VERB_MOVETO` 0 · `LINETO` 1 · `QUADTO` 2 · `CUBICTO` 3 · `CLOSE` 4
**Fill rules** `SD_FILL_EVEN_ODD` 0 · `SD_FILL_NONZERO` 1
**AA** `SD_AA_SUBSCANLINE` 0 · `SD_AA_AREA` 1
**Caps** `SD_CAP_BUTT` 0 · `ROUND` 1 · `SQUARE` 2
**Joins** `SD_JOIN_MITER` 0 · `ROUND` 1 · `BEVEL` 2
**Paint kinds** `SD_GRADIENT_LINEAR` 0 · `_RADIAL` 1 · `_FOCAL` 2 · `SD_PAINT_PATTERN` 3
**Spreads** `SD_SPREAD_PAD` 0 · `REPEAT` 1 · `REFLECT` 2 · `NONE` 3 *(patterns only)*
**Filters** `SD_FILTER_NEAREST` 0
**Polyline verdict** `SD_POLYLINE_TRUNCATED` 1 · `SD_POLYLINE_DEGRADED` 2
**Errors** `SADISH_OK` 0 · `_ERR_OOM` 1 · `_ERR_EMPTY_PATH` 2 · `_ERR_BOUNDS` 3 · `_ERR_UNSUPPORTED` 4 · `_ERR_OTHER` 99
**Sizes** `SD_BOUNDS_SIZE` 32 · `SD_POLYLINE_PT_SIZE` 16 · `SD_PATH_PT_SIZE` 16

---

## 5. A worked example

```
alloc_init();

var cv = sd_canvas_new(256, 256);
var out = sd_surface_new(256, 256);
if (cv == 0) { return 1; }                      # §2.3: test every constructor
if (out == 0) { return 1; }
sd_clear(out, sd_rgb(16, 16, 24));

var p = sd_path_new();
sd_path_moveto(p, 40 * SD_ONE, 40 * SD_ONE);
sd_path_cubicto(p, 200 * SD_ONE, 20 * SD_ONE,
                   60 * SD_ONE, 200 * SD_ONE,
                  210 * SD_ONE, 190 * SD_ONE);
sd_path_close(p);

var box = alloc(SD_BOUNDS_SIZE);                # conservative box, for culling
if (sd_path_bounds(p, box) == SADISH_OK) { ... }

sd_canvas_fill_path(cv, p, SD_FILL_NONZERO);    # geometry -> coverage

var g = sd_gradient_linear(40 * SD_ONE, 40 * SD_ONE, 210 * SD_ONE, 190 * SD_ONE);
sd_gradient_add_stop(g, 0,       sd_rgb(255, 80, 0));
sd_gradient_add_stop(g, SD_ONE,  sd_rgb(0, 120, 255));
sd_canvas_blit_paint_at(cv, out, g, 0, 0);      # coverage -> pixels

sd_surface_write_ppm(out, "out.ppm");
```

---

## 6. What this file does not cover

Internals (`_sd_*`), the coverage engine's accumulation, the flatten recursion, the dash walk, and
every measured figure. Those live in the `src/*.cyr` headers, which are long on purpose — each one
records *why* the code is shaped as it is and what breaks when it changes. Read them before editing.

Open questions and what 1.0 still needs: [`development/roadmap.md`](./development/roadmap.md).
What shipped when: [`../CHANGELOG.md`](../CHANGELOG.md).
