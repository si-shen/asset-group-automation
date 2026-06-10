---
name: asset-group-v1-formation
description: Form "Asset Group V1" from a spreadsheet of properties by concatenating a user-selected ordered set of attribute columns into a single new column called `Asset Group v1`. This is the FIRST step in a multi-step asset-grouping pipeline — v2, v3, v4 will be formed later with different logic and are handled by separate skills, so do NOT try to do them here. Use this skill whenever the user wants to create, generate, form, or define Asset Group V1 from property data (stock condition surveys, housing registers, portfolio lists, etc.), asks to "build asset group v1", wants a new column like `asset group v1` made by concatenating attributes (Local Authority, Property Type, Age Band, Construction Type, Tenure, Bedrooms, Region, Ward, etc.) with a pipe delimiter. Trigger even when the user doesn't use the exact phrase "asset group v1" — if they're concatenating chosen attribute columns into a new group-name column and no v2/v3/v4 logic has been applied yet, this is the skill.
---

# Asset Group V1 Formation

## What this skill does

Takes a spreadsheet of properties (one row per property, many attribute columns) and adds a single new column called **`asset group v1`** whose value is the concatenation of a user-chosen ordered set of attribute columns, delimited by `" | "`.

Example — from these row values:

| Local Authority | Property Type | Age Band    | Construction Type | Tenure |
|-----------------|---------------|-------------|-------------------|--------|
| Accrington      | H/B           | 1919 - 1944 | Trad              | GN     |

…the `asset group v1` value becomes:

```
Accrington | H/B | 1919 - 1944 | Trad | GN
```

The user picks **which** columns go in and **in what order**.

## Scope — v1 only

This skill forms **Asset Group V1 only**. It is the first stage of a multi-step asset-grouping pipeline:

- **v1** (this skill) — pure concatenation of user-selected attribute columns.
- **v2, v3, v4, …** — later stages that apply different logic (e.g. collapsing sparse groups, merging by rules, reclassifying against a lookup, minimum-group-size thresholds). These stages belong to **separate skills** and are out of scope here.

If the user asks for v2/v3/v4 behaviour while this skill is active, stop and tell them that v1 formation is complete and a different skill handles the next stage — don't improvise downstream logic inside this skill.

## Why people do this

In social housing and property portfolio management, "asset groups" (a.k.a. archetypes) are coarse buckets used for stock condition analysis, component life-cycle planning, decarbonisation modelling, and investment appraisal. V1 is the raw starting point — a faithful concatenation of whichever attributes the user considers defining. Later versions refine v1 using business rules, but those refinements only make sense on top of a clean v1, which is what this skill produces.

## The workflow (3 steps)

Always present the workflow as three clear steps so the user can follow along.

1. **Read the spreadsheet and list every column.** Pull the header row from the sheet (including duplicate header names — yes, real property sheets often have two columns both called "Property Type", one a shape category and one a form category, and both matter). Show the user a sample value from each column so they can tell them apart.
2. **Let the user pick the attributes in order.** The `asset group v1` name is a concatenation, so column *order* matters. Capture an ordered list, not an unordered set. Show a live preview built from the first row as soon as the user has selected at least one column, so they can see what a typical v1 group name will look like before committing.
3. **Generate and export.** Write the `asset group v1` value for every row, then save a new workbook alongside the original (don't overwrite) with that column appended as the last column. Report: total rows, number of unique v1 asset groups, and average properties per group — these stats tell the user immediately whether their chosen granularity is reasonable (one row per group = too fine; two groups covering thousands of rows = too coarse). The unique-count is also the handoff signal for v2: downstream skills typically kick in when v1 produces too many or too few groups to be useful as-is.

## Prefer the bundled HTML UI

There's a ready-made, self-contained browser tool at `assets/asset_group_builder.html`. It runs entirely client-side using SheetJS, walks the user through the three steps above, and exports an `.xlsx` or `.csv` with the `asset group v1` column appended. **This is the default deliverable.** Copy it into the user's workspace folder and give them a `computer://` link so they can open it in their browser.

Reasons to prefer the UI over writing a one-off Python script:

- The user can re-run it on new spreadsheets without calling back for help.
- They can try several column combinations in one session and see the preview change live — iterating on column choice is most of the actual work here.
- The unique-group-count stat updates instantly after each export, which is the feedback loop they need to pick the right granularity for v1 before v2/v3/v4 skills run on top of it.
- No upload of their data anywhere — it all stays in the browser.

Only fall back to a scripted approach (openpyxl/pandas) if the user explicitly wants batch automation, a CLI, a scheduled job, or the spreadsheet is too large for the browser (hundreds of thousands of rows).

## Column-selection rules

These come up every time. Bake them in so the user doesn't have to re-explain:

- **Order matters.** `Local Authority | Property Type | Age Band` and `Property Type | Local Authority | Age Band` are different v1 group names and different groupings downstream. Always capture order.
- **Duplicate headers are fine and often intentional.** The reference sheet has two columns literally named `Property Type` (e.g. one row reads `Bungalow` for the first and `H/B` for the second). Keep them distinct — don't dedupe header names.
- **Delimiter is `" | "`** (space-pipe-space). Don't substitute `-`, `_`, `/`, or a bare `|`. The spaces make the name legible in narrow table cells.
- **Blank values render as empty strings** and still take up their slot, so a row with a missing Age Band produces `Accrington | H/B |  | Trad | GN` (note the double-space). This is intentional for v1 — it keeps the column count stable and signals a data-quality gap rather than silently collapsing it. Handling missing values is a v2+ concern; don't impose a fix here unless the user explicitly asks.
- **Numbers come through as numbers**, not formatted strings. `No. Bedrooms` of `2.0` should render as `2`, not `2.0`. Strip trailing `.0` on integer-valued floats.
- **Trim whitespace** on string cell values — address and reference columns in these spreadsheets often have stray leading/trailing spaces from export pipelines.

## Output file conventions

- **Column name:** exactly `asset group v1` (lowercase, single space separators). Downstream v2/v3/v4 skills will key off this literal name, so don't rename it.
- **File name:** `<original-filename> - with asset group v1.xlsx`. Preserve the user's original file untouched.
- **Location:** alongside the original file in the user's workspace folder, so both are visible together.
- **Sheet name:** keep the original sheet name.
- **Column position:** append `asset group v1` as the rightmost column; don't reorder existing columns.
- **Keep all original rows and columns** — this is a non-destructive transformation. The output of v1 is the input to v2, so nothing can be dropped.

## Example interactions

**Example 1 — user asks for v1 explicitly**

> *User: "I've got this spreadsheet of 4,000 properties. Form asset group v1 based on local authority, property type, age band, construction, and tenure."*

Open the bundled HTML tool in their workspace, walk them through the three steps, and confirm the five columns they named (order: Local Authority → Property Type → Age Band → Construction Type → Tenure). Show them the first-row preview, e.g. `Coventry | H/B | 2000 - 2010 | Trad | GN`, then generate and link the exported file.

**Example 2 — user doesn't say "asset group v1" explicitly**

> *User: "Can you add a column that combines postcode, property type, and bedrooms with a pipe between them?"*

This is v1 formation — a concatenation of attribute columns into a derived label. Use the tool (or, if they explicitly want a script, openpyxl). Default the column name to `asset group v1` unless the user tells you otherwise.

**Example 3 — user asks for v2 behaviour mid-run**

> *User: "Now merge any group with fewer than 10 properties into a catch-all bucket."*

That's a v2 rule, not v1. Respond: "V1 is pure concatenation, so that rule belongs to the v2 formation step — I'll hand off to the v2 skill for that." Don't shoehorn the rule into this skill.

## What *not* to do

- **Don't do v2/v3/v4 logic here.** No minimum-group-size rules, no merging sparse groups, no lookup-based reclassification, no "UNKNOWN" substitution for blanks. V1 is a faithful concatenation; anything else is out of scope.
- **Don't invent categorical buckets of your own** (e.g. mapping `1919 - 1944` to "pre-war"). The user's columns are already the canonical vocabulary — your job is to concatenate, not to re-categorise.
- **Don't sort or deduplicate the rows.** The output has the same row count and row order as the input, with one extra column.
- **Don't lowercase, title-case, or otherwise normalise the values.** `H/B` and `h/b` mean the same thing to a human but they'll become different v1 groups if normalised inconsistently — safer to leave the raw values alone.
- **Don't drop columns the user didn't choose.** They stay in the sheet; they're just not part of the v1 concatenation. Later versions may need them.
