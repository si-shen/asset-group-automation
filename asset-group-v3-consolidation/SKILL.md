---
name: asset-group-v3-consolidation
description: Form "Asset Group V3" by consolidating small asset groups — for each group with fewer than a user-chosen threshold of properties, merge it with the best-matching adjacent group (same location/type/construction, adjacent age band, compatible tenure). This is the THIRD step in a multi-step asset-grouping pipeline; v1 formation and v2 split-block merging happen in separate skills and must already be done. Use this skill whenever the user wants to consolidate, reduce, shrink, or clean up small/tiny/sparse asset groups by merging them into neighbouring ones, set a minimum-group-size threshold, apply tenure-pair merge rules (e.g. AR + GN), merge across adjacent age bands (1980-1999 + 2000-2010), or "absorb overlapping tenures" (GN & AR absorbs GN). Trigger even when the user doesn't say "v3" — if they have an aggregated asset-group + property-count sheet and want to reduce the number of groups by merging undersized ones under rules that respect location, age band, and tenure, this is the skill. Do NOT trigger for pure concatenation (v1), block-reference split merging (v2), or lookup-based reclassification (v4+).
---

# Asset Group V3 Consolidation

## What this skill does

Takes an **aggregated** asset-group sheet (one row per group, with a property count) and merges small groups into adjacent ones, producing a shorter list of "Asset Group v3" names. The merge is rule-based: two groups can merge only if their non-age/tenure attributes match, their age bands are chronologically adjacent, and their tenures are compatible.

Example — from these input rows:

| Asset Group                                      | Properties |
|--------------------------------------------------|-----------:|
| Accrington \| H/B \| 1991 - 2000 \| Trad \| AR   |          1 |
| Accrington \| H/B \| 1991 - 2000 \| Trad \| GN   |          5 |
| Accrington \| H/B \| 2001 - 2010 \| Trad \| GN   |         31 |

…with a threshold of 5 and the pair `AR ↔ GN` declared, the first two rows consolidate:

| Original Asset Group                             | Properties | Merged Asset Group                                 | Merge Type            |
|--------------------------------------------------|-----------:|----------------------------------------------------|-----------------------|
| Accrington \| H/B \| 1991 - 2000 \| Trad \| AR   |          1 | Accrington \| H/B \| 1991 - 2000 \| Trad \| AR & GN | Small Group (< 5)     |
| Accrington \| H/B \| 1991 - 2000 \| Trad \| GN   |          5 | Accrington \| H/B \| 1991 - 2000 \| Trad \| AR & GN | Small Group (< 5)     |
| Accrington \| H/B \| 2001 - 2010 \| Trad \| GN   |         31 | *(unchanged)*                                       | *(unchanged)*         |

The third row stays put — it isn't small, and nothing smaller wanted to merge into it.

## Scope — v3 only

This skill forms **Asset Group V3 only**. It sits in the middle of a multi-step pipeline:

- **v1** — concatenate user-selected attribute columns into a single group name. Separate skill.
- **v2** — merge asset groups that share a physical block reference (split-block fix). Separate skill.
- **v3** (this skill) — consolidate small groups into adjacent ones using age-band and tenure rules.
- **v4, …** — later stages (e.g. lookup-based reclassification). Separate skills.

If the user asks for v1, v2, or v4+ behaviour while this skill is active, stop and tell them the relevant stage belongs to a different skill — don't improvise the other stages' logic inside this one.

## Why people do this

After v1 (concatenation) and v2 (block-split fixes), a real-world portfolio still ends up with hundreds of tiny asset groups: an Accrington semi-detached built 1991-2000 with AR tenure, with 1 property in it. At that granularity, stock-condition sampling, life-cycle modelling, and investment appraisal become statistically useless — you can't extrapolate from a sample of one. V3 consolidation folds these singletons into the nearest defensible neighbour (same location, type, construction; adjacent age band; compatible tenure) so every asset group has enough properties to be a useful analytical unit.

## The workflow (3 steps)

Always present the workflow as three clear steps.

1. **Upload the aggregated sheet.** One row per asset group, with a property count column. If the user only has a property-level file (one row per property, no count column), either hand off from the v2 skill — which aggregates automatically via its "Continue to V3" button — or tell them to pivot first. V3 will not re-aggregate arbitrary property-level sheets itself.
2. **Identify columns and split the name.** Ask for the asset-group column (auto-detect by header containing "asset group") and the count column (auto-detect by header containing "count" / "properties" or the first numeric column). Then split the first asset-group value by `" | "` and ask which piece is the **age band** and which is the **tenure**. These two positions drive every merge decision.
3. **Configure and consolidate.** Gather four pieces of configuration, then run:
   - **Threshold** (default 5): groups with fewer than this many properties are "small" and eligible to be merged.
   - **Tenure pairs**: which tenure codes may combine (e.g. `AR` ↔ `GN`). Merges only happen between tenures listed here, identical tenures, or inclusive tenures (see below).
   - **Allow pre/post-2000 merge** (default off): whether age bands straddling the year 2000 can merge.
   - **Also merge overlapping tenures** (default off): a second pass that absorbs strict-subset tenures (e.g. a `GN` group is absorbed into a `GN & AR` group with otherwise-identical attributes).

   Report: groups before / groups after / small groups before / small groups after / rows merged via threshold / rows merged via overlapping tenures. Offer an `.xlsx` download with the consolidated names and a reason column.

## Prefer the bundled HTML UI

There's a ready-made, self-contained browser tool at `assets/asset_group_builder.html`. It runs entirely client-side using SheetJS, exposes V1, V2, and V3 as tabs, and hands off between them. The V3 tab walks the user through the three steps above. **This is the default deliverable.** Copy it into the user's workspace folder and give them a `computer://` link so they can open it in their browser.

Reasons to prefer the UI over writing a one-off Python script:

- The threshold, tenure pairs, and toggles are iterative — the user will want to rerun several times with different settings to see how many groups remain. The UI re-runs instantly; a script is a round-trip.
- No upload of their data anywhere — it all stays in the browser.
- The V1 → V2 → V3 handoff buttons keep the pipeline in one place without the user re-uploading intermediate files.
- The bundled tool already matches the reference Streamlit implementation row-for-row, so there's no re-implementation risk.

Only fall back to a scripted approach (openpyxl/pandas) if the user explicitly wants batch automation, a CLI, or a scheduled job.

## Input data shape

V3 expects an **aggregated** sheet — one row per distinct asset group, with a property count. Not property-level.

- **Asset group column**: strings using `" | "` as the separator between attributes. The last two attributes are conventionally age band and tenure, but the user can pick any two positions.
- **Count column**: integer property counts per group. Floats and strings that parse as numbers are tolerated; missing/blank counts become 0 (and therefore "small").
- **Rows with missing asset group, tenure, or (if configured) age band are skipped** with a warning. This matches the Streamlit reference behaviour — without those fields the merge rules cannot be evaluated.

## Merge rules — the heart of V3

Two groups can merge only if **all** of these hold:

1. **Every non-age/tenure position matches exactly.** If group A is `Accrington | H/B | 1991-2000 | Trad | AR` and group B is `Accrington | F/M | 1991-2000 | Trad | AR`, they cannot merge — `H/B` ≠ `F/M` at the property-type position.
2. **Age bands are chronologically adjacent.** Parsed from year ranges, not string position — so `1965-1979`, `1965 - 1980`, `Pre 1919`, `Post 2020`, `1980 – 1999` (en-dash) all normalise correctly. Adjacent means one band ends where (or one year before where) the next begins; `1965-1980` is adjacent to both `1981-1990` and `1980-1999`. Two `Unknown` age bands are always adjacent. A single `Unknown` is adjacent to anything.
3. **The year-2000 boundary blocks adjacency unless explicitly allowed.** `1991-2000` and `2001-2010` are technically adjacent by years but will NOT merge by default — the 2000 boundary is a deliberate guardrail because many portfolios treat it as a significant building-regulations cutoff. The `Allow merging across the year-2000 boundary` toggle lifts this.
4. **Tenures are compatible.** Compatible means identical, in an inclusive relationship (see below), or in one of the user-declared tenure pairs. Tenure pairs are symmetric — declaring `AR ↔ GN` allows both directions.

### Tenure inclusivity

Tenures in this domain can be compound: `GN & AR` means the group holds both General Needs and Affordable Rent. `GN & AR` is **inclusive** of `GN` (every component of `GN` is in `GN & AR`). Inclusivity is checked by splitting on `&`, trimming, and testing set-inclusion — so `GN & AR` is inclusive of `GN`, of `AR`, and of `GN & AR` itself, but not of `MR`. When two groups have inclusive tenures and otherwise-identical attributes, the strictly-inclusive tenure "wins" for naming (no `&`-concatenation of already-overlapping terms).

### Conflict resolution

When a single small group has multiple valid merge partners, candidates are ranked by:

1. **Tenure-match score** (highest weight): inclusive > exact > paired.
2. **Age-band adjacency + preference score**: same band > adjacent > cross-2000 (when allowed).
3. **Smaller partner preferred** as a tiebreaker.

When multiple small groups want the same target, the **smallest small group wins** and each target is claimed at most once per pass. Left-over small groups with no available partner stay unmerged — that is expected; the report surfaces how many.

### Overlapping-tenure second pass (opt-in)

If the `Also merge overlapping tenures` toggle is on, a second pass runs after the threshold pass. For every pair of groups whose non-tenure attributes are identical and one tenure is a strict superset of the other, the subset is absorbed into the superset — regardless of property count. This is off by default because it's an O(n²) scan over the full portfolio and the user usually only wants it deliberately.

## Output file conventions

- **File name:** `<original-filename> - asset group v3.xlsx`. Preserve the user's original file untouched.
- **Sheet name:** `Consolidated`.
- **Columns, in order:** `Original Asset Group`, `Properties`, `Merged Asset Group`, `Merge Type`, `Age Band` (if an age-band position was picked), `Attribute N` for each non-age/tenure position (position-indexed names — so the skipped age-band position creates a gap like `Attribute 1, Attribute 2, Attribute 4`), `Tenure`.
- **Merged Asset Group and Merge Type are empty for rows that weren't touched.** Only rows whose consolidated name differs from the original get populated values. This makes the report scannable — filter by non-empty Merged Asset Group to see only the changes.
- **Merge Type** is either `Small Group (< N)` where N is the threshold, or `Inclusive Tenure` from the overlapping-tenure pass.

## Example interactions

**Example 1 — user asks for V3 explicitly**

> *User: "I've got this aggregated sheet of 1,400 asset groups. Consolidate any group under 5 properties by merging with adjacent groups. AR and GN can combine."*

Open the bundled HTML tool in their workspace, navigate to the V3 tab. Identify the asset-group and count columns. Split the first row and ask which piece is age band, which is tenure (the last two, typically). Set threshold to 5, add one tenure pair (`AR` ↔ `GN`), leave both toggles off, run. Report the before/after group count and download link.

**Example 2 — user doesn't say "v3" explicitly**

> *User: "Too many asset groups with only 1 or 2 properties. Merge the small ones into nearby ones if they're the same location and building type."*

This is V3. The "same location and building type" constraint is just the non-age/tenure-attributes-must-match rule — explain that the tool does this by default. Ask for their small-group threshold and which tenures they'd allow to combine. Don't silently assume AR + GN is OK; it isn't always.

**Example 3 — user expects V1 behaviour mid-run**

> *User: "Wait, can we rebuild the asset group name from scratch using different columns?"*

That's V1 formation, not V3. Respond: "Rebuilding the asset-group name from raw columns is V1 — the V1 tab will do that. V3 starts from an existing asset-group sheet and only renames merged pairs." Don't re-concatenate from raw attributes inside this skill.

**Example 4 — user wants the 2000 boundary lifted**

> *User: "Why didn't my 1991-2000 group merge with the 2001-2010 one? They're adjacent."*

They *are* adjacent by years, but V3 treats the year-2000 boundary as a default guardrail because of UK building-regulation shifts around then. Tell them there's a checkbox to lift it (`Allow merging across the year-2000 boundary`), and let them decide.

## What *not* to do

- **Don't do V1/V2/V4 logic here.** No concatenating from scratch, no block-reference merging, no lookup-based reclassification.
- **Don't merge groups whose non-age/tenure attributes differ.** A location change or a building-type change is never a valid V3 merge — even if the property count is tiny. The correct answer is to leave the group unmerged and surface it in the report.
- **Don't merge across the 2000 boundary unless the user opts in.** It's the default-off toggle for a reason.
- **Don't apply the overlapping-tenure pass by default.** It's an opt-in because it's slower and produces non-obvious merges that users sometimes don't want.
- **Don't claim a target group twice in the same pass.** One small group absorbs into one target; the target is then off the table for other small groups in that pass.
- **Don't drop unmerged rows from the output.** Every input row appears in the output, with the merged columns blank if no merge happened. This is a non-destructive transformation — the merged sheet is the full portfolio, just with a couple of added columns.
- **Don't normalise the asset-group strings.** Whatever case, dash style, or spacing was in the input goes back out unchanged (the merged name is built from the raw pieces). If the user's input has inconsistent `1965-1979` vs `1965 - 1980`, the age-band parser handles both for logic but the output preserves whatever the group originally said.
