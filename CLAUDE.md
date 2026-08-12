# CLAUDE.md

Guidance for Claude Code when working in this repository.

---

## What this project is

A governed metric dictionary for an **executive Talent Acquisition (TA) dashboard** — 15 metrics
covering hiring plan delivery, recruitment operations, funnel conversion, offer performance,
recruiter capacity, and hiring outcomes.

**Current focus: finalising the metric definitions themselves — one YAML file per metric.**
Everything else (gold ETL, semantic model, dashboard) comes later and reads from these files.

This is a portfolio and learning project. Prefer the simplest option that demonstrates the
concept clearly.

---

## The core rule: YAML is the source of truth

---

## Writing a metric definition

Follow `metric-definitions/_schema.yaml` exactly, **including key order** — it keeps diffs
readable and lets the set be validated as a group.

Rules that are easy to get wrong:

- `metric_id` is stable and never reused, even if the metric is deprecated.
- `measurement_window.rationale` is **required**. The window is where TA metrics usually go wrong,
  and the reasoning must outlast the person who wrote it.
- `precomputed_upstream` means the column is computed in the gold ETL, not the semantic model.
  If a value moves upstream, the gold contract must document it — the model can no longer see
  how it was derived.
- `related_metrics` should be reciprocal. If M04 lists M05, M05 lists M04.
- Formulas are **pseudocode, not DAX**. The dictionary describes intent; the model implements it.
- `notes` and `known_pitfalls` carry the rules that must survive translation into DAX. Keep them
  specific and actionable — not general advice.

---

## Style

Use clear, natural, professional English. Keep sentences short to medium in length and use plain business and technical language. Avoid academic or unnecessarily complex phrasing.

Explain a technical term briefly when it first appears. Do not oversimplify; keep explanations practical, precise, and technically accurate.

Always connect technical choices to the business decision or outcome they support. Do not explain a technical rule without explaining why it matters.
