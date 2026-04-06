# forecast-monitor-skill

An AI agent skill that turns a business metric forecast into a driver-level monitoring system, with sensitivity-derived alert thresholds, mix-shift decomposition, and on-demand replay of prior analyses.

---

## What it does

Given a forecast model (spreadsheet, doc, or structured data), an agent using this skill will:

- Extract the driver tree and classify each driver (core, phase, mix-shift, assumption)
- Derive alert thresholds from the model's own sensitivity analysis — high-elasticity drivers get tighter thresholds automatically
- Map each driver to a metric definition in a semantic layer, enforcing a single source of truth
- Generate a monitoring artefact suited to the available environment: BI dashboard, spreadsheet, alerting config, or SQL views
- Validate assumption consistency by surfacing when observed actuals imply a model assumption may no longer hold — without overriding it

## What it does not do

- Set or modify forecast assumptions — those remain human inputs
- Build the forecast model itself
- Prescribe a specific tool or environment

## Extensions

**Mix-shift monitoring** — for aggregate metrics (e.g. ARPU, margin rate) where composition change across segments is a distinct variance source from within-segment movement. Uses trend alerts rather than single-period thresholds.

**Analysis Lazy Reuse (Alert-Triggered)** — when a driver alert fires, checks whether a previously registered deep-dive analysis can explain the current variance. Runs only on alert; never on a schedule. Outputs `EXPLAINED`, `UNEXPLAINED`, or `NO_PRIOR_ANALYSIS` — never auto-diagnoses.

## Usage

This is a methodology skill for [Claude](https://claude.ai). Install the `.skill` file and invoke by describing a forecast model you want to turn into a monitoring system.

## Structure

```
forecast-monitor/
└── SKILL.md
```