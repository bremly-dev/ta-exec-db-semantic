# Executive Talent Acquisition Metric Dictionary

A governed set of metrics for an executive Talent Acquisition (TA) dashboard, designed to sit on
a gold-layer lakehouse and be served through a Power BI semantic model.

Portfolio project — the goal is to show how a TA Analytics team would define and govern metrics
before building the pipeline or dashboard, not to reproduce a full enterprise platform.

---

## The problem this solves

Most TA dashboards get built the other way round: someone drags fields into Power BI, and the
metric definitions get decided implicitly by whatever formula ends up in a DAX measure. That
works until two people report different "time to fill" numbers in the same meeting.

This project starts from the opposite direction: **agree the metric definitions first, in
writing.** Each metric states what it means, how it's calculated, what time window it uses and
why, which table it comes from, and how it tends to go wrong. That becomes the contract the data
model and dashboard get built against.

---

## What's inside

**15 metrics** across six areas:

| Area | Answers |
|---|---|
| Hiring Plan & Delivery | Are we delivering the plan, and what's about to slip? |
| Recruitment Operations | How fast are we, and where's the bottleneck? |
| Funnel & Conversion | Will we meet future demand, and why is the funnel leaking? |
| Offer Performance | Are we winning candidates, and do accepted offers stick? |
| Recruiter Capacity | Is the team sized and loaded correctly? |
| Hiring Outcomes | Are we hiring well, or just hiring fast? |

Hiring Outcomes (attrition, hiring manager satisfaction) is the deliberate counterweight — it
stops the other metrics from quietly rewarding "fast and cheap" over "good."

Some commonly tracked metrics — cost per hire, application volume, interview counts — were left
out on purpose, because they don't change an executive decision at this level.

---

## Repository structure

```
metric-definitions/
  _schema.yaml                    # shape every metric file follows
  metric_01_....yaml              # one file per metric, M01–M15
docs/
README.md
CLAUDE.md                         # working conventions for AI-assisted editing
```

Each metric is one YAML file and is the source of truth — formula, measurement window and its
rationale, source table, known pitfalls, semantic model notes. See `CLAUDE.md` for the convention.

---

## Status

Metric definitions are in progress, finalised one at a time. Data model, transformation layer,
and Power BI report come after the definitions are settled — by design.
