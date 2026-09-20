# sadish issues — what belongs here, and how to file one

Active filings live in this directory. Resolved ones move to [`archived/`](./archived/) and are
**never deleted** — a filing is the record of what was measured, and the measurement outlives the bug.
Proposals (capability requests rather than defects) live next door in
[`../proposals/`](../proposals/), with the same `archived/` rule.

Filenames stay stable across the move, so an `issues/<name>.md` reference in `src/`, `programs/` or
`CHANGELOG.md` gains one path segment — `issues/archived/<name>.md` — and nothing else
changes. ⛔ **Re-point every reference when you move a file.** `src/*.cyr` comments are copied verbatim
into `dist/sadish.cyr` by `cyrius distlib`, so a stale path there ships to every consumer; run
`cyrius distlib` after touching one. The sweep that proves it:

```sh
grep -rn "docs/development/" . --include='*.cyr' --include='*.md' --include='*.cyml'
```

Every path it prints must resolve to a file that exists. ⚠ Do not truncate a path with an ellipsis to
make a comment fit — `…-flatten-keeps-subdividing-…` resolves to nothing and hides from that grep.

## What belongs here

- **A defect a consumer hit** — rekha, dhancha, crab or agnos drawing through sadish and getting the
  wrong pixels, a fault, or a cost nobody budgeted for. Say which version you measured on.
- **A defect found in review** of sadish's own code, where the fix is bigger than the review.
- **A contract that two parts of sadish disagree about**, even when no caller can tell them apart
  today. [`archived/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md`](./archived/2026-09-14-direct-primitives-address-rows-by-width-not-stride.md)
  is the worked example: every packed surface rendered correctly, and the filing was still right.

## What does not belong here

- **A capability request.** That is a proposal — see
  [`../proposals/archived/2026-09-15-path-capacity-for-known-size-paths.md`](../proposals/archived/2026-09-15-path-capacity-for-known-size-paths.md).
- **A number you read rather than measured.** MEASURED means you ran it and can say with what.
- **Anything that fits in one line of `CHANGELOG.md`.** Fix it and write the line.

## How to file

Create a file in this directory named `YYYY-MM-DD-{kebab-case-slug}.md`. The slug is the FINDING, not the
area — `clip-masks-written-packed-but-read-by-canvas-stride`, not `clip-bug`. Header block:

```markdown
# {the finding, as a sentence}

**Status:** 🟡 **OPEN** — one line on what is still true.
**Filed:** YYYY-MM-DD, by {repo} ({what was being done}).
**Affects:** {versions}, and the FUNCTIONS involved.
**Severity:** {what breaks, for whom, today} — and what changes that.

## What happens / What was found
## MEASURED
## Suggested fix
```

⚠ **Cite functions, not `file:line`.** Line numbers rot within one release — the 0.7.1 audit found a
291-line drift on one site and all 16 references stale in another filing. Every filing here now
carries the re-derived lines under its own table; that is repair work nobody should have to repeat.

⭐ **MEASURED is the point.** `programs/*_test.cyr` is where a filing goes to stop being an opinion: a
closing claim is worth what its gate is worth, so say which suite, which checks, and what happens when
the fix is reverted one site at a time.

## Closing one

A filing closes when its **own asks** are met, not when the headline defect stops reproducing:

1. Rewrite the `**Status:**` line: 🟢, the version that shipped it, and the gate.
2. Add a closing section — what changed, the re-measured figures, and what is deliberately NOT done.
3. Add a `Closes` line to `CHANGELOG.md` naming the **archived** path, not the active one.
4. `git mv` it into [`archived/`](./archived/), re-point every reference, `cyrius distlib`.

⛔ **The status line is the part that goes wrong.** At the 0.7.1 audit, four of five filings here were
marked closed and three had unmet asks of their own — a truncated contour still filled open, a stroke
that still faults on a refused allocation, arrays not opened at the capacity the proposal asked for.
Two named a release, `0.6.1`, that was never cut. All three are still in this directory with corrected
status lines and a **Still open** section, and they are the worked examples of the failure mode:
*archiving is how sadish asserts something is done.* Do not archive a filing to tidy the directory.
