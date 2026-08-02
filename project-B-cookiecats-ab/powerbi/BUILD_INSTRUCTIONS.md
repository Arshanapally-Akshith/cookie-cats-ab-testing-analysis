# Build Instructions — Cookie Cats A/B Test Power BI Report

Target: one page, 1280×720, light theme, ~20–30 minutes to build. Every number on
the page must trace back to the notebook — the checklist at the end verifies that.

**Design principle (revised):** every chart and KPI in this version is bound
directly to a pre-shaped CSV — long/tidy for charts, one wide row for the KPI
cards. This replaces the earlier approach of rebuilding rates/deltas/text via
DAX (or unpivoting `cookie_cats` in Power Query for the retention chart) with a
plain drag-and-drop field mapping. `DAX_measures.md` still exists and is still
useful — as an **independent cross-check** computed live from `cookie_cats`, to
prove the CSV values weren't mistyped — but it is no longer required to build
the base report.

Files in this folder:
- `data/cookie_cats.csv` *(referenced from `../data/raw/`, not copied)* — raw
  row-level data. Used only for the live cross-check measures, not by any visual
  directly.
- `data/kpi_cards.csv` — **new.** One row, wide. Every KPI card value, caption,
  and badge text, pre-formatted. Drives all 5 KPI cards.
- `data/retention_chart.csv` — tidy long format (`metric`, `group`,
  `retention_rate`, as decimal fractions). Drives the retention clustered column
  directly — no unpivot needed.
- `data/guardrail_chart.csv` — **new.** `group`, `median_rounds`. Drives the
  guardrail column chart directly.
- `data/sample_allocation.csv` — `group`, `n_players`, `pct_actual`,
  `pct_expected` (percentage columns now stored as decimal fractions). Drives
  the sample-split stacked bar.
- `data/stats_summary.csv` — the three pre-specified test results, plus two
  pre-formatted columns (`ci_range`, `significant_label`) added to remove the
  Power-Query text-concatenation step the matrix used to need. Drives the
  Statistical Summary matrix.
- `data/srm_test.csv` — SRM chi-square result + Phase 1 data-quality note.
  Unchanged — already fraction-scaled (p-values, not percentages).
- `DAX_measures.md` — optional cross-check measures, see note above.

> **Note on this revision:** three files that already existed
> (`retention_chart.csv`, `sample_allocation.csv`, `stats_summary.csv`) were
> edited — the first two to convert percentage-style numbers (`44.82`) to
> decimal fractions (`0.4482`) so Power BI's built-in Percentage format works
> without a workaround, and the third to add two computed-once text columns.
> No statistical value was changed in any of them — see `DATA_MODEL.md` for the
> full column-level diff. `srm_test.csv` and `cookie_cats.csv` are untouched.

---

## Step 1 — New report & get data

1. Power BI Desktop → **Blank report** → save as `cookiecats_ab_dashboard.pbix` in this `powerbi/` folder.
2. **Home → Get Data → Text/CSV**, add all six sources (Import mode, not DirectQuery):
   - `../data/raw/cookie_cats.csv` → table name **`cookie_cats`**
   - `data/kpi_cards.csv` → table name **`kpi_cards`**
   - `data/retention_chart.csv` → table name **`retention_chart`**
   - `data/guardrail_chart.csv` → table name **`guardrail_chart`**
   - `data/sample_allocation.csv` → table name **`sample_allocation`**
   - `data/stats_summary.csv` → table name **`stats_summary`**
   - `data/srm_test.csv` → table name **`srm_test`**
3. Click **Transform Data** before loading — go to Step 2 first.

## Step 2 — Power Query: types only, no transforms

No unpivoting, no calculated columns, no text concatenation anywhere in this
version — every table loads and is used exactly as shaped in the CSV. Just
confirm types:

- `cookie_cats`: `userid`/`sum_gamerounds` → Whole Number; `version` → Text;
  `retention_1`/`retention_7` → **True/False (Boolean)** (needed only for the
  optional DAX cross-check measures — see `DAX_measures.md`).
- `kpi_cards`: the five `*_delta_value` columns and `total_players` → Decimal
  Number / Whole Number; everything else (`*_caption`, `*_badge_text`,
  `*_badge_status`, `recommendation_value`, `recommendation_caption`) → Text.
- `retention_chart`: `metric`/`group` → Text; `retention_rate` → **Decimal
  Number**, then set its default format to **Percentage, 1 decimal** in the
  Report view (Column tools → Format → Percentage) so the fraction displays
  correctly everywhere it's used.
- `guardrail_chart`: `group` → Text; `median_rounds` → Whole Number.
- `sample_allocation`: `group` → Text; `n_players` → Whole Number;
  `pct_actual`/`pct_expected` → **Decimal Number**, format **Percentage**.
- `stats_summary`: `rate_control`/`rate_treatment`/`abs_diff`/`ci_low`/`ci_high`
  → Decimal Number (leave unformatted as plain numbers — see the per-row unit
  caveat below); `rel_diff_pct`/`p_value`/`test_stat`/`effect_size` → Decimal
  Number; `ci_range`/`significant_label`/`significant`/`test_name`/
  `effect_size_name`/`metric`/`group_control`/`group_treatment` → Text.
- `srm_test`: `chi2_stat`/`p_value`/`threshold` → Decimal Number; rest → Text/
  Whole Number as named.

**Close & Apply.**

## Step 3 — Model view

Open **Model view**. You should see 6 tables (7 once you add the optional
`_Measures` table) and **no relationship lines** — this is correct. Every
reference table (`kpi_cards`, `retention_chart`, `guardrail_chart`,
`sample_allocation`, `stats_summary`, `srm_test`) is a standalone, pre-shaped
table consumed directly by one visual. Do not draw relationships between any of
them or to `cookie_cats`.

**Optional — for validation only:** if you want the live cross-check measures,
**Home → Enter Data**, create an empty table named `_Measures`, and paste in the
measures from `DAX_measures.md`. Skip this if you just want the report built
quickly; do it if you want a second, independently-computed set of numbers to
diff against the CSVs (recommended before an interview, not required to render
the page).

## Step 4 — Colors

No theme file is imported in this build — an earlier custom theme JSON didn't
pass validation, so colors are applied manually per visual instead (each step
below states its exact hex values: blue `#2A78D6` / orange `#EB6834`
categorical, plus the good/neutral/bad status colors). This is slightly more
manual than a one-click theme import, but it's what's actually reflected in the
shipped report, so build it this way rather than trying to reintroduce a theme
file. **Light theme only** — no dark-mode toggle on this report.

## Step 5 — Page setup

**Format page → Canvas settings**: Type = **Custom**, Width **1280**, Height
**720**. Background solid `#FCFCFB` (set manually via Format page →
Canvas background). Turn off page scrolling.

| Zone | y-range | Contents |
|---|---|---|
| Header band | 0–64 | Title, subtitle, version slicer, meta text |
| KPI row | 72–182 | 5 KPI cards |
| Chart row | 192–422 | Retention chart · Guardrail chart · Experiment Quality Check panel |
| Detail row | 434–644 | Statistical summary matrix · Recommendation callout |
| Footer | 656–686 | Source citation |

Use **Format → Align/Distribute** for pixel-perfect spacing within each row.

---

## Step 6 — Header band

1. **Text box**, x=24 y=12 w=700 h=28: **"Cookie Cats A/B Test Analysis"** —
   Segoe UI Semibold, 20pt, `#14140F`.
2. **Text box**, x=24 y=40 w=700 h=20: **"Impact of Gate Placement on Player
   Retention (90,189 Players)"** — Segoe UI, 11pt, `#52514A`.
3. **Slicer**, x=940 y=10 w=210 h=32 — field: `cookie_cats[version]`. Style
   **Tile**, orientation **Horizontal**.
4. **Text box**, x=940 y=44 w=210 h=16: *"n = 90,189 · single randomized
   experiment window"* — 9pt, `#8A8879`.
5. **Critical — Edit Interactions.** Select the slicer → **Format → Edit
   interactions**. Set interaction to **None** for every visual built from
   `kpi_cards`, `stats_summary`, `sample_allocation`, and `srm_test` — these are
   fixed, pre-computed results and must not change if a viewer filters to a
   single group. Only the retention chart, guardrail chart, and any live
   cross-check measures (if you built `_Measures`) should respond to the
   slicer.

## Step 7 — KPI row (5 cards, left → right)

All cards: **Card** visual, background `#F6F5F1`, 1px border `#1C000000`,
radius 4px. **Data source: `kpi_cards` (single row) for every field below —
drag the column straight onto the Card's Fields well.** No measures required.

1. **Players Analyzed** — x=24 w=228
   - Value: `kpi_cards[total_players]`
   - Caption text box: `kpi_cards[players_split_caption]`
   - No badge (neutral informational card)

2. **Day-1 Retention Δ** — x=266 w=228
   - Value: `kpi_cards[day1_delta_value]` — format **Percentage, 2 decimals,
     show sign** (Format → Values → Display units: None; use a custom format
     string `+0.00%;-0.00%` for the leading sign)
   - Caption text box: `kpi_cards[day1_caption]`
   - Badge text box: `kpi_cards[day1_badge_text]` — muted gray pill (this row's
     `day1_badge_status = "muted"`)

3. **Day-7 Retention Δ** — x=508 w=228 — **accented card**: background
   `#FBE9E8`, left border 3px `#B23434`
   - Value: `kpi_cards[day7_delta_value]`, same percentage format as above,
     28pt font (larger than the other cards — this is the headline finding)
   - Caption: `kpi_cards[day7_caption]`
   - Badge: `kpi_cards[day7_badge_text]` — red pill (`day7_badge_status =
     "critical"`)

4. **Guardrail: Median Rounds Δ** — x=750 w=228
   - Value: `kpi_cards[guardrail_delta_value]` — format **Whole Number, show
     sign** (`+0;-0`), suffix " round" via custom format if desired
   - Caption: `kpi_cards[guardrail_caption]`
   - Badge: `kpi_cards[guardrail_badge_text]` — green pill
     (`guardrail_badge_status = "good"`, framed as "no negative impact," per
     the notebook's own framing)

5. **Recommendation** — x=992 w=264 — background `#14140F` (inverted), text
   white — a *decision*, not a measurement, styled distinctly on purpose
   - Value: `kpi_cards[recommendation_value]` ("Keep Level 30")
   - Caption: `kpi_cards[recommendation_caption]`, small, muted white

> `*_badge_status` columns (`muted`/`critical`/`good`) exist so you can drive
> badge pill color with **conditional formatting by field value** instead of
> manually recoloring each badge — Format → Conditional formatting → Background
> color → Format by: Field value → bind to the matching `*_badge_status`
> column, then map `muted → #F6F5F1`, `critical → #FBE9E8`, `good → #E6F6E6`.

## Step 8 — Chart row, Panel A: Retention by gate placement

x=24 y=192 w=520 h=230. **Clustered column chart.**
- **Data: `retention_chart`** (4 rows, already tidy — no Power Query step
  needed)
  - **Axis:** `retention_chart[metric]` (Day-1 Retention, Day-7 Retention)
  - **Legend:** `retention_chart[group]` (Gate 30, Gate 40)
  - **Values:** `retention_chart[retention_rate]`
- Data labels: On, **Percentage, 1 decimal** (the fraction values format
  natively — no custom measure needed).
- Colors: Gate 30 = `#2A78D6`, Gate 40 = `#EB6834`, set explicitly per series
  in Format → Colors (not left to theme auto-assignment).
- Y-axis: 0–50%, gridlines on, hairline `#E1E0D9`.
- Title: **"Retention by Gate Placement"**. Subtitle text box: *"% of players
  returning on Day-1 and Day-7"*.
- Two small text boxes as significance flags above each category: bind to
  `kpi_cards[day1_badge_text]` (muted) and `kpi_cards[day7_badge_text]` (red) —
  same source as the KPI badges, so the flag text can never drift from the
  card above it.

## Step 9 — Chart row, Panel B: Gameplay engagement

x=556 y=192 w=340 h=230. **Clustered column chart.**
- **Data: `guardrail_chart`** (2 rows)
  - **Axis:** `guardrail_chart[group]`
  - **Values:** `guardrail_chart[median_rounds]`
- Data labels: On, whole number.
- Colors: same fixed blue/orange mapping as Panel A.
- Title: **"Gameplay Engagement"**. Subtitle: *"Median rounds played — checks
  the retention change isn't masking a volume shift"*.
- Caption text box below the chart, bound to `kpi_cards[guardrail_pvalue_
  caption]` ("Mann-Whitney U · p = 0.050 (n.s.)").

## Step 10 — Chart row, Panel C: Experiment Quality Check

x=908 y=192 w=348 h=230. Bundles sample allocation + SRM + a data-quality note.

- Title: **"Experiment Quality Check"**. Subtitle: *"Confirms the comparison
  above is trustworthy before reading it"*.
- **100% stacked bar chart**, top of panel — **Data: `sample_allocation`**
  - **Axis:** none needed (single constant category) — or add a literal
    "Split" text column if the visual requires an axis field; alternatively use
    a simple **Stacked bar chart** with Axis = a blank/constant.
  - **Legend:** `sample_allocation[group]`
  - **Values:** `sample_allocation[pct_actual]` — format **Percentage, 1
    decimal**; the fractions (0.4956 / 0.5044) now render correctly without
    any extra scaling.
  - Add a **constant analytics line at 0.5** (Analytics pane → Constant line →
    value 0.5 — matches the fraction scale) to show deviation from the
    expected split.
- Below the bar, three text elements, **Data: `srm_test`** (1 row — bind
  directly to columns, no measure needed):
  - `srm_test[p_value]` + `srm_test[verdict]`, e.g. via a text box reading
    "SRM check (χ²): p = " next to a Card bound to `srm_test[p_value]`
    (format **Number, 4 decimals** — this is a p-value, not a percentage, so
    do **not** apply Percentage format here), followed by
    `srm_test[verdict]`.
  - Sample split readout: reuse the stacked bar's own data labels — no
    separate text needed.
  - Data-quality note: text box bound to `srm_test[note]`, smaller muted
    italic text.
- This panel must **never respond to the version slicer** (Step 6.5).

## Step 11 — Detail row: Statistical summary matrix

x=24 y=434 w=760 h=210. **Table** visual, **Data: `stats_summary`**, columns in
this exact order:
`metric` · `rate_control` · `rate_treatment` · `abs_diff` · `rel_diff_pct` ·
`ci_range` · `p_value` · `significant_label`.

- `ci_range` is now a **plain text column already formatted per row**
  (`"[-1.24pp, +0.06pp]"` for the retention rows, `"[-1, 0] rounds"` for the
  guardrail row) — the Power-Query text-concatenation step from the previous
  build is no longer needed.
- `significant_label` is now a **plain text column already worded correctly
  per row** ("Not significant" / "Significant decline" / "No negative
  impact") — the custom calculated-column icon-mapping step is no longer
  needed. If you still want an icon, add conditional formatting on the
  `significant` (Yes/No) column instead: icon set, red down-triangle for
  "Yes" on the two retention rows, green check for "No" on the guardrail row —
  set this as **rules per row value of `metric`**, not a single global rule,
  since "Yes" means "bad" for retention but "No" means "good" for the
  guardrail.
- `rate_control`/`rate_treatment`/`abs_diff` formatting: apply **Percentage**
  format to the Day-1 and Day-7 rows' cells conceptually, but remember this is
  one column spanning all three rows — because the guardrail row holds a raw
  round count (17/16/−1) in the same column, do **not** apply a blanket
  Percentage format to the whole column, or the guardrail row will render as
  "1700%"/"1600%". Format the column as a plain **Decimal Number**, and rely on
  `ci_range`/`significant_label` (already unit-correct text) plus the visual's
  subtitle to communicate units, or split the guardrail row into its own small
  card instead of the shared table if you want native percentage formatting on
  the retention rows.
- Title: **"Statistical Summary"**. Subtitle: *"Full detail behind the KPI
  cards"*.
- Turn off slicer interaction (Step 6.5).

## Step 12 — Detail row: Recommendation callout

x=796 y=434 w=460 h=210.
- Container: rectangle shape, background `#F6F5F1`, left border 3px `#B23434`,
  corner radius 0/5/5/0.
- Eyebrow text box: *"RECOMMENDATION"*, 10pt bold uppercase, `#B23434`.
- Heading: a borderless **Card** visual, **Data: `kpi_cards[recommendation_
  value]`**, large text — single source of truth with the Recommendation KPI
  card in Step 7 (both point at the same CSV cell, so they can never disagree).
- Detail line: text box or small Card bound to `kpi_cards[recommendation_
  caption]`.
- Bullet list (static text box — this is prose, not tabular data, so a CSV
  would add friction rather than remove it):
  - "Day-7 retention drops ~4.3% at Level 40 — small, but statistically
    significant (p = 0.0016)."
  - "No offsetting gain in gameplay volume — the guardrail metric shows no
    benefit to compensate for the retention cost."
  - "Effect size is small: a caution against shipping, not evidence of a
    severe problem."
  - "Caveat: borderline sample-ratio mismatch (p = 0.0086) — confirm
    randomization before finalizing."
- Footer line, small muted text: *"Full methodology in
  cookiecats_ab_analysis.ipynb."*
- Turn off slicer interaction (Step 6.5).

## Step 13 — Footer

x=24 y=660 w=1232 h=24, two text boxes, 9pt `#8A8879`:
- Left: *"Source: Kaggle — Cookie Cats (Tactile Entertainment) · n = 90,189"*
- Right: *"Analysis: cookiecats_ab_analysis.ipynb · Prepared for: Product /
  Growth"*

---

## Final QA — before calling it done

- [ ] KPI row numbers match Phase 2/3 notebook output exactly (cross-check
      against `DAX_measures.md`'s validation table if you built `_Measures`)
- [ ] Retention chart bars match 44.82%/44.23% and 19.02%/18.20% — check the
      Percentage format didn't get applied twice (e.g. showing "4482.00%")
- [ ] Guardrail chart bars match median 17/16
- [ ] Experiment Quality Check shows p = 0.0086 and the correct 49.6%/50.4%
      split (confirm this reads as a percentage, not "49.56%%" or "0.50")
- [ ] Matrix table matches `stats_summary.csv` row for row, and the guardrail
      row's numeric column does **not** render as a percentage
- [ ] Recommendation card and callout both say "Keep Level 30" and pull from
      the same `kpi_cards` cell
- [ ] Slicer does **not** change the matrix, quality-check panel, KPI row, or
      recommendation card when set to a single group
- [ ] No visual anywhere reports a mean-based `sum_gamerounds` comparison as
      the headline guardrail number (median only)
- [ ] Title reads exactly **"Cookie Cats A/B Test Analysis"**, subtitle
      exactly **"Impact of Gate Placement on Player Retention (90,189
      Players)"**
- [ ] Only the light theme is active

## Export for the portfolio

**File → Export → PDF** for a static leave-behind. Keep the `.pbix` itself as
the interactive artifact for a live interview walkthrough.
