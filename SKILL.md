---
name: forecast-monitor
description: >
  Use this skill when the user wants to turn a business metric forecast or financial model into
  a living monitoring system — one where actuals are automatically compared against forecast
  drivers, variances surface at the driver level (not just output), and alerts fire based on
  sensitivity-derived thresholds. Trigger whenever the user has a forecast model (spreadsheet,
  doc, structured data) and wants to: (1) build a variance monitoring system from it, (2) define
  driver-level alerts, (3) reduce the manual work of reconciling actuals vs forecast, or (4)
  avoid rebuilding dashboards every time the model changes. Also trigger for phrases like
  "model-driven monitoring", "forecast to monitoring pipeline", "driver-based alerts",
  "automate variance analysis", or "connect my model to a BI tool".
---

# Forecast-to-Monitor Skill

## Purpose

Given a business metric forecast (revenue, GMV, DAU, etc.) built on an explicit driver structure,
this skill guides an agent to construct a **model-driven monitoring system** that:

- Tracks actuals against forecast at the **driver level**, not just the output metric
- Derives alert thresholds from the model's own sensitivity analysis
- Validates assumption consistency (without replacing the human's job of setting them)
- Produces monitoring artefacts appropriate to the available environment

This is a **methodology skill** — it is tool- and environment-agnostic. The agent selects
output form (BI dashboard, spreadsheet, alerting config, SQL views, etc.) based on what is
available and what the user needs.

---

## Core Concepts

### 1. The Driver-Output Hierarchy

Every forecast metric is a function of drivers:

```
Output Metric  =  f(Driver₁, Driver₂, ... Driverₙ)

Example:
  Revenue  =  Users  ×  ARPU
  Users    =  Users_prev  +  New_users  −  Churned_users
  ARPU     =  Avg_price  ×  Avg_usage_per_user
```

Monitoring **only** the output metric means variance attribution requires manual re-derivation
every cycle. The goal is to make driver-level variance visible automatically, so the output
variance is already explained when it lands.

### 2. Driver Classification

Before building any monitoring artefact, classify every driver in the model:

| Class | Definition | Monitoring approach |
|---|---|---|
| **Core driver** | Structural; present across all business phases (e.g. users, ARPU, churn) | Always monitored; stable definition |
| **Phase driver** | Stage-specific; valid for current business model only (e.g. a single paid channel's CAC) | Monitored while active; replaceable without touching core |
| **Mix-shift driver** | Aggregate metric movements caused by segment composition change, not within-segment change | Monitored as structural drift; trend alert, not threshold alert (see below) |
| **Assumption** | Human-set input; intentional, documented, not derived from data | Not monitored as a live metric; validated for consistency only |
| **Output** | Derived result of drivers × assumptions | Monitored as a lagging signal; explained by drivers |

**Assumptions are human territory.** The monitoring system does not track assumption values
as live metrics. It may validate that actuals imply an assumption violation (e.g. "observed
ARPU is inconsistent with the pricing assumption"), but it does not attempt to auto-update
or override assumptions.

### 3. The Semantic Layer Dependency

This skill assumes a **metric definition layer** exists or will be created — a single source
of truth for how each driver is computed from raw data. This might be:

- A dbt metrics layer
- A shared SQL view library
- A named range or formula definition sheet (in a spreadsheet context)
- A config file consumed by a BI tool

**Every driver in the monitoring system must resolve to exactly one definition in the semantic
layer.** If two definitions exist for the same driver, resolve the conflict before proceeding.
Ambiguous definitions produce unreliable variance, which defeats the purpose.

### 4. Mix-Shift Drivers

Mix shift is a distinct variance source: an aggregate metric (e.g. ARPU, margin rate, CAC)
moves not because any segment changed, but because the **composition of segments changed**.
It must be decomposed separately from within-segment movement, otherwise the two effects
mask each other.

**Decomposition structure:**

```
Aggregate metric M  =  Σᵢ (share_i × M_i)

Variance of M  =  within-segment effect  +  mix-shift effect

  within-segment effect  =  Σᵢ share_i_forecast × (M_i_actual − M_i_forecast)
  mix-shift effect       =  Σᵢ (share_i_actual − share_i_forecast) × M_i_forecast
```

**When to classify a driver as mix-shift:**

- The model contains segment-level breakdowns of an aggregate metric, OR
- The user explicitly needs to monitor structural composition drift, AND
- The semantic layer has segment-level definitions available

If neither condition holds, do not force mix-shift decomposition — monitor the aggregate
driver as a core driver and note that unexplained residuals may have mix-shift origins.

**Monitoring logic for mix-shift drivers differs from standard drivers:**

| Aspect | Standard driver | Mix-shift driver |
|---|---|---|
| Alert type | Drift alert (single-period threshold) | Trend alert (N consecutive periods moving same direction) |
| Threshold basis | Elasticity-derived % threshold | Directional: share moving consistently toward low-value segments |
| Frequency | Per driver's data cadence | Same cadence, but evaluated as a rolling window |
| Message content | Variance + implied output impact | Which segment is growing/shrinking + implied ARPU / margin drag |

**Driver inventory entry for mix-shift:**

```
  - driver: arpu_mix_shift
    class: mix-shift
    segments: [enterprise, smb, consumer]
    segment_definitions: [semantic_layer_ref]
    data_source: known
    frequency: monthly
    elasticity_to_output: medium
    alert_type: trend
    trend_window: 3          # flag if mix drifting same direction for 3+ periods
```

---

## Agent Workflow

### Phase 0: Parse the Forecast Model

Before building anything, extract the model's structure:

1. **Identify the target output metric(s)** — what the model is ultimately forecasting
2. **Extract the driver tree** — the full decomposition from output to leaf drivers
3. **Separate drivers from assumptions** — drivers have data sources; assumptions are
   human-set parameters (growth rates, market share targets, pricing decisions)
4. **Note the time segmentation** — which drivers have per-period granularity vs. terminal
   assumptions; this determines monitoring frequency per driver
5. **Extract any existing sensitivity analysis** — which drivers have elasticity estimates
   relative to the output metric

If the model does not have explicit sensitivity analysis, prompt the user or perform a
simple one-factor sensitivity pass before proceeding to Phase 2.

**Output of Phase 0**: A structured driver inventory (see template below).

```
Driver Inventory Template
─────────────────────────
metric_name:      [e.g. Monthly Revenue]
output_metric:    [e.g. revenue_usd]
driver_tree:
  - driver: new_users
    class: core
    data_source: [known / unknown / TBD]
    frequency: weekly
    elasticity_to_output: high   # from sensitivity analysis
  - driver: churn_rate
    class: core
    data_source: [known / unknown / TBD]
    frequency: biweekly
    elasticity_to_output: high
  - driver: avg_price
    class: core
    data_source: [known / unknown / TBD]
    frequency: monthly
    elasticity_to_output: medium
  - driver: paid_channel_cac
    class: phase
    data_source: [known / unknown / TBD]
    frequency: weekly
    elasticity_to_output: low
assumptions:
  - name: market_growth_rate
    value: 12%
    rationale: [as documented in model]
  - name: pricing_uplift_y2
    value: 5%
    rationale: [as documented in model]
```

---

### Phase 1: Define Alert Thresholds from Sensitivity

This is the critical step that connects the forecast model to the monitoring system —
**thresholds are derived from elasticity, not set arbitrarily**.

**Principle**: A driver with high elasticity to the output warrants a tighter alert threshold
and higher monitoring frequency. A low-elasticity driver can tolerate larger variance before
action is warranted.

**Threshold derivation logic**:

```
For each driver d with elasticity e_d (defined as % change in output / % change in d):

  acceptable_output_variance_budget  =  T   (e.g. 5% tolerable output miss)
  driver_threshold(d)  =  T / e_d

  Example:
    T = 5%,  e_churn = 8x  →  churn_threshold = 5% / 8 = 0.6pp
    T = 5%,  e_price = 2x  →  price_threshold = 5% / 2 = 2.5%
```

If elasticity values are ordinal (high/medium/low) rather than numeric, use the following
defaults as a starting point:

| Elasticity | Alert threshold | Monitoring frequency |
|---|---|---|
| High | ±5% of forecast (or ±0.5pp for rates) | Weekly or daily |
| Medium | ±10–15% of forecast | Biweekly |
| Low | ±20–25% of forecast | Monthly |

These defaults should be overridden with numeric elasticity wherever it exists.

**Alert types**:

- **Drift alert**: driver is outside threshold for the current period
- **Trend alert**: driver has moved in the same direction for N consecutive periods
  (catches slow deterioration that single-period thresholds miss)
- **Assumption violation signal**: actual implied value of an assumption is inconsistent
  with the model's stated assumption (e.g. implied churn rate from actuals is 4.2% vs
  model assumption of 2.5%); this is surfaced as a flag, not an alert — it informs the
  human that an assumption may need revisiting

---

### Phase 2: Map Drivers to Data Sources

For each core and phase driver, resolve its data source in the semantic layer:

```
For each driver in the inventory:
  1. Confirm the definition exists in the semantic layer
  2. If it does not exist: flag it and surface the definition needed
  3. If multiple definitions exist: flag the conflict and pause until resolved
  4. Note update cadence of the underlying data source
     (this caps the monitoring frequency — can't monitor weekly if data is monthly)
```

**Do not proceed to Phase 3 until every monitored driver has exactly one resolved definition.**
Unresolved drivers should be listed explicitly so the user can action them.

---

### Phase 3: Design the Monitoring Artefact

The form of the monitoring system depends on the available environment. Select or combine:

#### Option A — BI Dashboard Layer

When a BI tool (Looker, Metabase, Tableau, etc.) is available:

- Create one view per driver that joins forecast values to actuals
- Add a variance column: `(actual - forecast) / forecast`
- Add a threshold flag column: `abs(variance) > threshold → TRUE`
- Compose a summary view that shows: output metric variance, then driver variances ranked
  by elasticity (highest impact drivers first)
- Alerts can be configured as BI tool notifications or scheduled report thresholds

Structure the dashboard so **output metric is the last thing shown**, not the first.
The driver panel should explain the output variance before the user even reads the output row.

#### Option B — Spreadsheet / Google Sheets

When operating in a spreadsheet environment:

- One tab: **Forecast** (human-maintained, locked for editing except assumption cells)
- One tab: **Actuals input** (data entry or auto-populated via connector)
- One tab: **Variance** (computed, never manually edited)
  - Column structure: `driver | forecast | actual | variance_abs | variance_pct | threshold | flag`
  - Rows sorted by elasticity descending
  - Conditional formatting: flag cells red when `abs(variance_pct) > threshold`
- One tab: **Assumption consistency** (derived checks, e.g. implied churn vs assumed churn)

The Forecast tab is the single source of both the driver tree and the threshold values.
When the model is updated, only the Forecast tab changes; Variance and Assumption tabs
recompute automatically.

#### Option C — Alerting / Notification System

When the goal is proactive notification rather than a dashboard:

- Define alerts as structured config (YAML, JSON, or equivalent):

```yaml
alerts:
  - driver: churn_rate
    elasticity: high
    threshold_pct: 0.6      # in percentage points for rates
    frequency: weekly
    channel: slack           # or email, pagerduty, etc.
    message_template: >
      Churn rate actual ({actual}%) is {variance}pp vs forecast ({forecast}%).
      At current elasticity, this implies ~{output_impact}% revenue miss.

  - driver: new_users
    elasticity: high
    threshold_pct: 5
    frequency: weekly
    channel: slack
    message_template: >
      New user adds ({actual}) are {variance_pct}% vs forecast ({forecast}).
```

- Alert messages should always include the **implied output impact**, not just the driver
  variance. This makes the alert actionable without the recipient having to re-derive it.

#### Option D — SQL Views + Scheduled Query

When operating in a data warehouse environment:

```sql
-- Pattern for each driver
CREATE OR REPLACE VIEW monitor_driver_churn AS
SELECT
  period,
  forecast_churn_rate,
  actual_churn_rate,
  (actual_churn_rate - forecast_churn_rate)                AS variance_pp,
  ABS(actual_churn_rate - forecast_churn_rate) > 0.006     AS alert_flag,  -- threshold from elasticity
  (actual_churn_rate - forecast_churn_rate) * 8            AS implied_rev_impact_x  -- elasticity = 8
FROM forecast_actuals_churn;

-- Summary view: all drivers, ranked by elasticity
CREATE OR REPLACE VIEW monitor_summary AS
SELECT 'churn_rate'  AS driver, 'high'   AS elasticity, variance_pp AS variance, alert_flag, implied_rev_impact_x FROM monitor_driver_churn
UNION ALL
SELECT 'new_users'   AS driver, 'high'   AS elasticity, ...
UNION ALL
SELECT 'avg_price'   AS driver, 'medium' AS elasticity, ...
ORDER BY
  CASE elasticity WHEN 'high' THEN 1 WHEN 'medium' THEN 2 ELSE 3 END,
  ABS(variance) DESC;
```

Scheduled queries run on the driver's monitoring frequency; a downstream alert job reads
the summary view and notifies on flagged rows.

---

### Phase 4: Rolling Forecast Integration

The monitoring system must not be static. When the forecast model is updated:

1. **Driver tree changes**: If a new driver is added or a phase driver is retired, update
   the driver inventory and re-derive thresholds. The semantic layer definition for any
   new driver must be resolved before it enters the monitoring system.

2. **Assumption changes**: Assumption updates do not require monitoring system changes.
   They do require re-running the sensitivity analysis if the elasticity structure may
   have changed materially (e.g. the business model shifted).

3. **Threshold recalibration**: Thresholds derived from elasticity should be reviewed
   whenever the model's sensitivity analysis is updated — not on a fixed calendar.

4. **Model retirement**: When the business model changes significantly (new revenue stream,
   new pricing structure), the old model should be formally retired rather than patched.
   The monitoring system for the old model stays live but is labelled historical.
   A new driver inventory is created for the new model.

---

### Phase 5: Assumption Consistency Validation

This is distinct from driver monitoring. The goal is to surface signals that a model
assumption may no longer hold — not to replace it.

For each assumption in the model, define a corresponding observable:

```
Assumption: churn_rate = 2.5% (model input)
Observable: implied_churn = churned_users / users_prev_period  (from actuals)
Validation: IF implied_churn deviates from assumption by > X for Y consecutive periods
            → surface flag: "Observed churn (3.8%) has diverged from model assumption
              (2.5%) for 3 consecutive months. Consider revisiting this assumption."
```

This validation output is a **flag for human review**, not an automated correction.
The flag should always include: observed value, assumed value, duration of divergence,
and a plain-language description of what this implies for the forecast.

---

## Output Checklist

Before delivering any monitoring artefact, verify:

- [ ] Every monitored driver resolves to exactly one definition in the semantic layer
- [ ] Thresholds are derived from sensitivity / elasticity, not set arbitrarily
- [ ] Output metric variance is explained by driver variances (no unexplained residual
      larger than rounding/data-lag)
- [ ] Mix-shift drivers (if present) use trend alerts, not single-period thresholds
- [ ] Mix-shift decomposition separates within-segment from composition effects
- [ ] Assumption cells / parameters are clearly separated and not auto-updated by the system
- [ ] Assumption consistency checks are present for every major model assumption
- [ ] Alert messages include implied output impact, not just driver variance
- [ ] The system has a defined update procedure when the forecast model changes
- [ ] Phase drivers are clearly labelled as replaceable; core drivers are clearly stable

---

## Extension: Analysis Lazy Reuse (Alert-Triggered)

This extension is **non-standard and optional**. It applies when a driver has previously
been the subject of a one-off deep-dive analysis that decomposed it further than the
regular monitoring layer — and that decomposition logic should be replayable on demand
when the same driver fires an alert again.

**It does not run on a schedule. It only activates when a driver alert fires.**

### When to use this extension

- A driver alert has fired in the past, and a human analyst performed a manual
  deep-dive to explain it (e.g. churn spike traced to cohort X + region Y)
- The decomposition logic from that analysis is available in a reusable form
  (SQL, notebook, named calculation, annotated breakdown)
- The same driver has now fired again, and the question is: does the prior
  decomposition explain the current variance, or is this a new pattern?

### What it does

On alert trigger for a registered driver:

1. **Check the analysis registry** — does a stored decomposition exist for this driver?
2. **If yes**: replay the decomposition logic against current-period actuals
3. **Evaluate explanatory power**: does the replayed decomposition account for
   ≥ X% of the observed variance? (X is user-defined; 80% is a reasonable default)
4. **Output one of two results**:
   - `EXPLAINED`: prior decomposition accounts for the variance →
     surface the breakdown with current values, flag it as a known pattern
   - `UNEXPLAINED`: prior decomposition does not account for the variance →
     flag for new manual investigation; do not attempt to auto-diagnose

The extension never auto-diagnoses unexplained variance. Its job is to quickly
rule in or rule out a known pattern, not to replace human analysis.

### Analysis registry structure

Store each registered decomposition as a reusable entry:

```yaml
analysis_registry:
  - driver: churn_rate
    alert_id: churn-2024-q2          # human-readable reference to original analysis
    decomposition_dimensions:
      - cohort_month
      - acquisition_channel
      - region
    logic_ref: sql/churn_deep_dive.sql   # or notebook path, named view, etc.
    explanatory_threshold: 0.80          # flag as EXPLAINED if decomp accounts for ≥80% variance
    created: 2024-07-15
    created_by: [analyst name / team]
    notes: >
      Original spike was concentrated in month-3 cohorts from paid social,
      APAC region. Logic isolates this cut; if pattern recurs it should show
      up in the same dimensions.

  - driver: arpu_mix_shift
    alert_id: arpu-mix-2024-q3
    decomposition_dimensions:
      - segment
      - product_tier
    logic_ref: sql/arpu_mix_breakdown.sql
    explanatory_threshold: 0.75
    created: 2024-10-02
    created_by: [analyst name / team]
```

### Trigger and output format

```
ON alert_fired(driver: churn_rate):
  IF driver IN analysis_registry:
    replay(logic_ref)
    compute explanatory_power = variance_explained / total_variance

    IF explanatory_power >= explanatory_threshold:
      OUTPUT:
        status: EXPLAINED
        pattern_ref: churn-2024-q2
        current_breakdown: [replayed decomposition with current values]
        variance_explained: 87%
        note: "Prior pattern (month-3 cohorts, paid social, APAC) accounts
               for 87% of current variance. Review breakdown before escalating."

    ELSE:
      OUTPUT:
        status: UNEXPLAINED
        pattern_ref: churn-2024-q2
        variance_explained: 31%
        note: "Prior decomposition explains only 31% of current variance.
               This appears to be a new or different pattern. Manual
               investigation recommended."
  ELSE:
    OUTPUT:
      status: NO_PRIOR_ANALYSIS
      note: "No registered decomposition for this driver. Manual
             investigation required. Consider registering findings
             for future lazy reuse."
```

### Registry maintenance

- Entries are added manually after a human analyst completes a deep-dive
- Entries are not auto-generated by the monitoring system
- An entry should be **retired** when the business context has changed enough
  that the original decomposition dimensions are no longer meaningful
  (e.g. a region was merged, a product tier was discontinued)
- Retired entries are archived, not deleted — historical context has value

### What this extension does not do

- Run decompositions on a schedule
- Auto-generate new decomposition logic
- Propose explanations for UNEXPLAINED variance
- Update or replace the registered decomposition logic automatically

---

## What This Skill Does Not Do

- **Set assumptions** — growth rates, market share targets, pricing decisions remain
  human inputs. The skill may surface that an assumption appears inconsistent with
  observed data, but it does not propose new assumption values.
- **Build the forecast model** — this skill takes a model as input. Model construction
  (driver selection, assumption rationale, scenario design) is outside scope.
- **Choose a BI tool** — the skill is tool-agnostic. It produces the monitoring logic;
  implementation in a specific tool follows that tool's own patterns.
- **Guarantee forecast accuracy** — the monitoring system surfaces variance quickly and
  at the right level. It does not improve the model's predictive quality.