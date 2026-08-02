# DAX Measures — Cookie Cats A/B Test Dashboard

Design principle: anything **trivially and exactly reproducible** (counts, rates,
medians, simple differences) is computed live in DAX from `cookie_cats`, so it can
never silently drift from the raw data. Anything that required an actual
statistical library in the notebook (p-values, confidence intervals, z/U
statistics, effect sizes — Mann-Whitney U, bootstrap CIs, Welch's t-test) is
**imported as-is** from `stats_summary` / `srm_test`, not re-derived in DAX. DAX
has no rank-based or resampling test, so approximating one would be a silent
re-analysis, which the project's own rule forbids ("Power BI is presentation, not
re-analysis").

All measures below assume table names `cookie_cats`, `stats_summary`,
`sample_allocation`, `srm_test` exactly as created in Step 2 of
`BUILD_INSTRUCTIONS.md`. Create a dedicated **measure table** (Step 3) and put all
of these there rather than scattering them across the data tables.

---

## 1. Core counts (from `cookie_cats`)

```dax
Total Players = COUNTROWS(cookie_cats)

Players Gate 30 = CALCULATE([Total Players], cookie_cats[version] = "gate_30")

Players Gate 40 = CALCULATE([Total Players], cookie_cats[version] = "gate_40")

Players Gate 30 % = DIVIDE([Players Gate 30], [Total Players])

Players Gate 40 % = DIVIDE([Players Gate 40], [Total Players])
```

## 2. Retention rates (from `cookie_cats`)

`retention_1` / `retention_7` must be typed as **True/False (Boolean)** in Power
Query (see Build Instructions Step 2) — `AVERAGE` on a boolean column returns the
proportion of TRUE, which is exactly the retention rate.

```dax
Day-1 Retention Rate = AVERAGE(cookie_cats[retention_1])

Day-7 Retention Rate = AVERAGE(cookie_cats[retention_7])

Day-1 Retention (Gate 30) = CALCULATE([Day-1 Retention Rate], cookie_cats[version] = "gate_30")

Day-1 Retention (Gate 40) = CALCULATE([Day-1 Retention Rate], cookie_cats[version] = "gate_40")

Day-7 Retention (Gate 30) = CALCULATE([Day-7 Retention Rate], cookie_cats[version] = "gate_30")

Day-7 Retention (Gate 40) = CALCULATE([Day-7 Retention Rate], cookie_cats[version] = "gate_40")

Day-1 Retention Δ (pp) = ([Day-1 Retention (Gate 40)] - [Day-1 Retention (Gate 30)]) * 100

Day-7 Retention Δ (pp) = ([Day-7 Retention (Gate 40)] - [Day-7 Retention (Gate 30)]) * 100

Day-1 Retention Δ (%) = DIVIDE([Day-1 Retention (Gate 40)] - [Day-1 Retention (Gate 30)], [Day-1 Retention (Gate 30)])

Day-7 Retention Δ (%) = DIVIDE([Day-7 Retention (Gate 40)] - [Day-7 Retention (Gate 30)], [Day-7 Retention (Gate 30)])
```

**Cross-check after building:** `[Day-1 Retention (Gate 30)]` unfiltered must read
**44.82%**, `[Day-7 Retention (Gate 30)]` must read **19.02%** — these should match
the notebook's Phase 2 output exactly (proportions are exact arithmetic; there is
no room for rounding drift). If they don't match, the boolean typing or the
`version` text values (case/whitespace) are wrong.

## 3. Guardrail metric (from `cookie_cats`)

```dax
Median Rounds Played = MEDIAN(cookie_cats[sum_gamerounds])

Median Rounds (Gate 30) = CALCULATE([Median Rounds Played], cookie_cats[version] = "gate_30")

Median Rounds (Gate 40) = CALCULATE([Median Rounds Played], cookie_cats[version] = "gate_40")

Median Rounds Δ = [Median Rounds (Gate 40)] - [Median Rounds (Gate 30)]

Median Rounds Δ (%) = DIVIDE([Median Rounds Δ], [Median Rounds (Gate 30)])

Never-Started Rate = DIVIDE(
    CALCULATE(COUNTROWS(cookie_cats), cookie_cats[sum_gamerounds] = 0),
    [Total Players]
)
```

**Do not** add a DAX "average rounds played" measure for headline reporting — the
notebook deliberately rejected the mean here (a single 49,854-round outlier
distorts it). If you want a mean available for drill-through/tooltip context only,
label it explicitly "Mean (not outlier-robust)" so it can't be mistaken for the
metric the test conclusion is based on.

## 4. Inferential statistics (imported from `stats_summary`)

```dax
Day-1 p-value =
CALCULATE(
    SELECTEDVALUE(stats_summary[p_value]),
    stats_summary[metric] = "Day-1 Retention"
)

Day-7 p-value =
CALCULATE(
    SELECTEDVALUE(stats_summary[p_value]),
    stats_summary[metric] = "Day-7 Retention"
)

Guardrail p-value =
CALCULATE(
    SELECTEDVALUE(stats_summary[p_value]),
    stats_summary[metric] = "Guardrail - Median Rounds"
)

Day-1 Significant = CALCULATE(SELECTEDVALUE(stats_summary[significant]), stats_summary[metric] = "Day-1 Retention")

Day-7 Significant = CALCULATE(SELECTEDVALUE(stats_summary[significant]), stats_summary[metric] = "Day-7 Retention")

Guardrail Significant = CALCULATE(SELECTEDVALUE(stats_summary[significant]), stats_summary[metric] = "Guardrail - Median Rounds")
```

Badge/label text measures (used on KPI cards, driven by the imported p-values so
the wording can never contradict the number next to it):

```dax
Day-1 Badge Text = IF([Day-1 Significant] = "Yes", "Significant", "Not significant · p = " & FORMAT([Day-1 p-value], "0.000"))

Day-7 Badge Text = IF([Day-7 Significant] = "Yes", "Significant decline · p = " & FORMAT([Day-7 p-value], "0.000"), "Not significant")

Guardrail Badge Text = IF([Guardrail Significant] = "Yes", "Significant change", "No reliable impact · p = " & FORMAT([Guardrail p-value], "0.000"))
```

## 5. Experiment quality (imported from `sample_allocation` / `srm_test`)

```dax
SRM p-value = SELECTEDVALUE(srm_test[p_value])

SRM Chi-Square = SELECTEDVALUE(srm_test[chi2_stat])

SRM Verdict = SELECTEDVALUE(srm_test[verdict])

Data Quality Note = SELECTEDVALUE(srm_test[note])
```

Sample allocation is consumed directly from `sample_allocation[pct_actual]` on the
stacked bar's Values well — no measure required, but for the readout text:

```dax
Sample Split Label =
"Gate 30: " & FORMAT(CALCULATE(SUM(sample_allocation[pct_actual]), sample_allocation[group] = "Gate 30"), "0.0") & "%" &
"   ·   Gate 40: " & FORMAT(CALCULATE(SUM(sample_allocation[pct_actual]), sample_allocation[group] = "Gate 40"), "0.0") & "%"
```

## 6. Recommendation KPI card

Kept as measures (not hardcoded text-box copy) so the whole report has exactly one
place to edit if the recommendation is ever revisited:

```dax
Recommendation = "Keep Level 30"

Recommendation Detail = "Day-7 retention −4.3% at Level 40 (p = 0.0016) · no guardrail benefit"
```

## 7. Formatting helpers (optional, used on KPI cards)

```dax
Day-1 Retention Δ (pp, formatted) = FORMAT([Day-1 Retention Δ (pp)], "+0.00;-0.00") & " pp"

Day-7 Retention Δ (pp, formatted) = FORMAT([Day-7 Retention Δ (pp)], "+0.00;-0.00") & " pp"

Median Rounds Δ (formatted) = FORMAT([Median Rounds Δ], "+0;-0") & " round"
```

---

## Validation checklist — every measure against the notebook

| Measure | Expected value | Notebook source |
|---|---|---|
| `[Total Players]` | 90,189 | Phase 1 §3.1 |
| `[Players Gate 30]` / `[Players Gate 40]` | 44,700 / 45,489 | Phase 1 §4 |
| `[Day-1 Retention (Gate 30)]` / `(Gate 40)` | 44.82% / 44.23% | Phase 2 §2.1 |
| `[Day-7 Retention (Gate 30)]` / `(Gate 40)` | 19.02% / 18.20% | Phase 2 §2.2 |
| `[Day-7 p-value]` | 0.0016 | Phase 2 §2.2 (imported, not recomputed) |
| `[Median Rounds (Gate 30)]` / `(Gate 40)` | 17 / 16 | Phase 3 §3.4 |
| `[Guardrail p-value]` | 0.0502 | Phase 3 §3.4 (Mann-Whitney, imported) |
| `[SRM p-value]` | 0.0086 | Phase 1 §5 |

If any live-computed measure (rates, counts, medians) doesn't match its notebook
value exactly, the fix is in the data/model — never adjust the measure to force a
match.
