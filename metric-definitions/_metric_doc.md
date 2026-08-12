# Executive Talent Acquisition Metric Dictionary

**Purpose:** A governed set of 15 metrics for an executive TA dashboard, sized to be built on a gold-layer lakehouse and served through a Power BI semantic model.

**Design principle:** Every metric below earns its place by changing a decision. Metrics that are interesting but not actionable at executive level (source of applicants, interview volume, cost per hire in isolation) are deliberately excluded and noted at the end.

---

## A. Hiring Plan & Delivery

### 1. Offer Plan Attainment (To Date)

**Business definition**
How much of the hiring plan has actually been delivered so far this year, measured at accepted offer, compared to how much *should* have been delivered by this point. This is the headline number of the dashboard.

**Formula / calculation logic**
```
Offer Plan Attainment % = Actual Offers Accepted To Date / Planned Offers To Date
```
- "Actual" is counted at **accepted offer**, not start date — accepted offer is the earlier, more controllable signal, and it's the one this whole metric family is built around. Actual start can be shown as a secondary reconciliation view.
- "Planned to date" comes from the phased hiring plan (planned offers spread across months), not the full-year number divided by twelve — plans are rarely flat.
- Grain: business unit × job family × month, rolled up.
- Support a **To Date / Full Year** toggle, since executives ask both questions in the same meeting.

**Suggested gold table**
`fct_offer` (one row per accepted offer) joined to `fct_hiring_plan` (plan grain: business unit × job family × month) via `gold_dim_date`, `gold_dim_org`, `gold_dim_job_family`.

**Executive rationale**
Answers the first question any executive asks: *are we delivering what we committed to?* It is also the number finance and business leaders already track, so aligning TA to it removes arguments about whose number is right. Anchoring on offer accepts rather than starts keeps the number timely — starts lag offers by weeks and would make this headline stale.

---

### 2. Forecasted Attainment (Next End of Month)

**Business definition**
Where offer accepts are projected to stand at the end of the *current* month if the pipeline and current pace continue. This is a short-horizon forecast — the number leadership can sanity-check against what they already see in the pipeline — rather than a full-year projection.

**Formula / calculation logic**
```
Days Remaining = Last Day of Current Month − Today

Forecast Offers (Next EOM) = Offers Accepted To Date (this month)
                           + (Open Reqs × Expected Offer-Accept Probability within Days Remaining)

Forecasted Attainment % (Next EOM) = Forecast Offers (Next EOM) / Planned Offers for Current Month
```
- Probability should be scoped to the **days remaining in the month**, not a generic full-cycle probability — a req with 35 days of typical time-to-offer-accept and 6 days left in the month has a low probability of landing this month even if it will very likely land next month. Use historical offer-accept-rate-by-elapsed-time curves per job family, not a flat rate.
- Show the forecast as a **range** (conservative / expected) if the data supports it. A single point estimate invites false precision.
- Because the horizon is short, this metric should refresh and be discussed **weekly** — it's the rolling "are we on pace to hit this month" check, distinct from the full-year attainment trend.

**Suggested gold table**
`fct_requisition` (open req state, target hire date, stage) + `fct_offer` + `fct_hiring_plan` (monthly grain), with offer-accept-rate-by-elapsed-time curves from a small governed `gold_ref_fill_benchmark` table.

**Executive rationale**
Attainment to date is history and cannot be changed. A next-end-of-month forecast is the nearest-term number leadership can still act on this week — nudging a stuck req, pulling forward an offer, adding sourcing help — before the month closes. It converts the meeting from a status report into an immediate planning conversation, and it's easier to trust than a full-year projection because it's checkable against the pipeline leadership can see right now.

---

### 3. Offers at Risk

**Business definition**
The number of open requisitions projected to miss their target hire date — i.e. unlikely to reach an accepted offer in time — and the offers those requisitions represent.

**Formula / calculation logic**
```
For each open requisition:
  Projected Offer-Accept Date = Today + Expected Remaining Days (based on current stage and job family)
  At Risk = Projected Offer-Accept Date > Target Hire Date
```
```
Offers at Risk = COUNT(open reqs where At Risk = TRUE)
```
- Expected remaining days should come from historical median stage durations for that job family, not a single global average.
- Break out by business unit, job family, and a **critical role** flag so leadership sees *where* the risk sits.
- Optional severity banding (at risk / severely at risk) based on how many days past target.

**Suggested gold table**
`fct_requisition` joined to `fct_application_stage` for current stage, with benchmarks from `gold_ref_stage_benchmark`.

**Executive rationale**
This is the metric that generates the action list. It answers *what is about to go wrong, and where do I need to intervene now?* — while there is still time to add sourcing support, reprioritise, or reset expectations with the business.

---

## B. Recruitment Operations

### 4. Median Time to Fill (Mix-Adjusted)

**Business definition**
How long it typically takes to fill a role, from requisition approval to offer acceptance — adjusted so that a change in the *type* of roles being hired does not look like a change in team performance.

**Formula / calculation logic**

The day count is **already computed upstream** and lands in the gold layer as a column on the offer fact. The semantic model consumes it directly and does not recompute it from dates.

```
-- Upstream (gold ETL, not the semantic model):
time_to_fill_days  →  precomputed column on fct_offer
                      (offer accepted date − requisition approved date)

-- Semantic model:
Median Time to Fill = MEDIAN( fct_offer[time_to_fill_days] )
                      FILTERED TO accepted offers only
                      over the selected period
```
Mix adjustment (calculated in the model, since it depends on the selected period):
```
Mix-Adjusted TTF = SUM( Median time_to_fill_days by job family × Baseline mix weight )
```
- **Filter to accepted offers.** The offer fact carries extended, declined, and withdrawn offers too. Only accepted offers have a meaningful time-to-fill; leaving the others in would count roles that were never filled.
- **Watch for multiple offers on one requisition.** If a candidate declines and a second offer is extended to someone else, that requisition produces two offer rows. Only the accepted one should reach this measure — otherwise a single hard-to-fill role contributes several data points and drags the median. The accepted-offer filter handles this in most cases, but confirm the upstream job doesn't leave more than one accepted offer per requisition for standard (single-hire) reqs.
- Use **median**, not average. One 200-day executive search will distort an average and start the wrong conversation.
- Because the value is precomputed, the semantic model's job is aggregation and weighting only. Do **not** rebuild the date subtraction in DAX — two versions of the same calculation is how the number starts disagreeing with itself.
- **Medians do not pre-aggregate.** A precomputed row-level `time_to_fill_days` is exactly what's needed; a precomputed *median* at some fixed grain is not, because it cannot be rolled up correctly when the user changes filters. Keep the day count at offer grain and let the model take the median.
- Baseline mix weights should be fixed (for example, prior full-year role mix) so period-over-period comparisons are fair.
- Also publish the raw (unadjusted) median as a secondary figure — some stakeholders will want it, and hiding it damages trust.

**Suggested gold table**
`fct_offer` — one row per offer, carrying the precomputed `time_to_fill_days` on accepted offers, plus `offer_status`, `requisition_id`, `job_family`, and `complexity_tier`. Weights from `gold_ref_role_mix_baseline`.

Sourcing this from the offer fact keeps the whole delivery-and-speed family (metrics 1, 2, 3, 4) reading from the same population of accepted offers, so "how many did we deliver" and "how fast did we deliver them" can never disagree about which records count.

Three things the upstream job must define and document, since the semantic model can no longer see them:
- **Clock start.** Requisition approved date, not posting date or created date. State which, because the difference is often two weeks.
- **Clock pauses.** Whether on-hold or cancelled-then-reopened periods are excluded from the day count. If they are not excluded, a req put on hold for a month will look like a slow fill.
- **Multi-hire requisitions.** For evergreen or volume reqs that fill several seats, decide whether each accepted offer carries its own day count from the single requisition approval date. It usually should — but note the tenth hire on an evergreen req will show a very long time-to-fill through no fault of the recruiter, so these reqs may warrant exclusion from the headline.

**Executive rationale**
Speed is the main operational lever TA controls. Mix adjustment prevents the most common false alarm in TA reporting — the team looks slower simply because the business asked for harder roles this quarter. Computing the day count upstream also means every consumer — this dashboard, the recruiter scorecard, any downstream AI agent — reads the same governed number rather than each re-deriving it.

---

### 5. Median Time in Stage

**Business definition**
How long candidates sit in each step of the process — sourcing, screening, hiring manager review, interview, offer.

**Formula / calculation logic**
```
Time in Stage = Stage Exit Timestamp − Stage Entry Timestamp
Median Time in Stage = MEDIAN by stage, by job family, by period
```
- Calculate on candidates who have **exited** the stage, and show a separate "currently waiting" figure for candidates still sitting in it. The waiting figure is often the more urgent one.
- Attribute each stage to its owner (recruiter-owned vs hiring-manager-owned) — this is what makes the metric actionable.

**Suggested gold table**
`fct_application_stage` (candidate × requisition × stage, with entry and exit timestamps).

**Executive rationale**
"Time to fill went up" is not actionable. "Hiring manager review is taking nine days instead of three" is. This metric locates the bottleneck and, critically, shows when the delay sits with the business rather than with TA — which is often the single most valuable insight in the weekly meeting.

---

## C. Funnel & Conversion

### 6. Pipeline Coverage Ratio

**Business definition**
How much live, late-stage candidate pipeline exists relative to the roles that need filling. The earliest reliable warning that future delivery will miss.

**Formula / calculation logic**
```
Pipeline Coverage = Active Candidates at Interview Stage or Later / Open Requisitions
```
- Calculate by job family and business unit, not just in total — a healthy overall ratio can hide a critical function with no pipeline at all.
- Set a target ratio from your own historical conversion (for example, if roughly one in four onsite candidates is hired, a coverage ratio below 4:1 signals shortfall).
- Count only **active** candidates; withdrawn and rejected must be excluded or the metric silently inflates.

**Suggested gold table**
`fct_application_stage` (current active stage per candidate) + `fct_requisition` (open reqs).

**Executive rationale**
Delivery and time-to-fill are lagging. Coverage is leading — it tells leadership about next quarter's shortfall *this* quarter, when sourcing investment can still change the outcome.

---

### 7. Stage Conversion Rate (Pass-Through)

**Business definition**
The percentage of candidates who move from one stage of the process to the next.

**Formula / calculation logic**
```
Stage Conversion % = Candidates Entering Next Stage / Candidates Entering Current Stage
```
- Measure on a **cohort** basis: track candidates who entered a stage in a given period through to their outcome. Measuring entries and exits in the same calendar window produces misleading numbers when volumes are changing.
- Key ratios for the executive view: application → screen, screen → interview, interview → offer.
- Show against a rolling prior-period benchmark, not an industry figure. Internal trend is far more reliable than external comparison.

**Suggested gold table**
`fct_application_stage`, aggregated to a cohort-based `fct_funnel_conversion` if performance requires pre-aggregation.

**Executive rationale**
Conversion diagnoses *why* the funnel is underperforming. A weak application-to-screen rate is a sourcing quality problem. A weak interview-to-offer rate is usually a calibration problem between recruiter and hiring manager. These require completely different interventions, and only this metric tells them apart.

---

### 8. Source Effectiveness (Hire Yield by Source)

**Business definition**
Which sourcing channels actually produce hires — not which produce the most applicants.

**Formula / calculation logic**
```
Source Hire Yield % = Hires from Source / Candidates Entering Funnel from Source
Source Share of Hires = Hires from Source / Total Hires
```
- Pair yield with volume. A channel with 40% yield and four candidates is not a strategy.
- Use **first-touch** or **last-touch** attribution consistently and state which on the page. Mixing the two is the most common source of disputes about these numbers.
- Referrals, agency, direct sourcing, job boards, and internal mobility are the groupings that matter at executive level.

**Suggested gold table**
`fct_application` joined to `gold_dim_source`, rolled up to `fct_source_performance`.

**Executive rationale**
This is a budget metric. It answers *where should we spend the next sourcing dollar, and which agency or job board contract should we not renew?* It is also the natural response when pipeline coverage is low.

---

## D. Offer Performance

### 9. Offer Acceptance Rate

**Business definition**
The percentage of offers extended that candidates accept.

**Formula / calculation logic**
```
Offer Acceptance Rate = Offers Accepted / Offers Extended (same offer cohort)
```
- Use a **cohort denominator**: offers extended in the period, tracked to their eventual outcome. Do not divide accepts in a period by extends in that period — offers span periods and the ratio becomes meaningless.
- Exclude offers still pending decision from the denominator, or show them separately as "undecided". Be explicit about which choice you made.
- Segment by job family, level, and business unit. A blended 88% can hide a 60% rate in engineering.
- Attach **decline reasons** (compensation, counteroffer, location, competing offer, timing) as a drill-through.

**Suggested gold table**
`fct_offer` (one row per offer with extended / accepted / declined / withdrawn status, decision date, decline reason).

**Executive rationale**
A falling acceptance rate is usually a market signal, not a recruiting failure — compensation bands slipping, a slow process losing candidates to faster competitors, or a weakening employer brand. It is one of the few TA metrics that regularly triggers action *outside* TA, in compensation or business leadership.

---

### 10. Offer Renege & Rescind Rate

**Business definition**
The percentage of accepted offers that never turn into a start — either the candidate withdrew after accepting (renege) or the company withdrew the offer (rescind).

**Formula / calculation logic**
```
Renege Rate = Accepted Offers that did not start (candidate-initiated) / Accepted Offers
Rescind Rate = Accepted Offers withdrawn by company / Accepted Offers
```
- Track these **separately**. They have opposite causes and opposite owners.
- Measure on a cohort with a completed observation window (for example, offers accepted at least 60 days ago), so recent accepts do not artificially depress the rate.
- Important governance note: because reneges and rescinds arrive in later data refreshes, **hire counts will restate between refreshes**. Surface a small "as of" note and a restatement indicator on the page rather than explaining this verbally every week.

**Suggested gold table**
`fct_offer` joined to `fct_hire` (start confirmation), with a `fct_offer_status_history` for the restatement audit trail.

**Executive rationale**
This is the gap between the number TA reports and the number the business feels. A team can hit accepted-offer plan and still miss headcount if reneges run high. It also protects the credibility of the whole dashboard — leadership discovering restatement on their own is far more damaging than TA disclosing it up front.

---

## E. Recruiter Capacity & Throughput

### 11. Recruiter Requisition Load (Complexity-Weighted)

**Business definition**
How many open requisitions each recruiter is carrying, weighted so that a hard-to-fill senior role counts for more than a high-volume entry-level role.

**Formula / calculation logic**
```
Weighted Load = SUM(Open Reqs × Complexity Weight) per recruiter
```
- Complexity weights should come from a governed reference table (for example: volume role = 0.5, standard professional = 1.0, senior / specialist = 2.0, executive = 3.0), calibrated from historical effort or time to fill.
- Report distribution (median, and count of recruiters above the healthy threshold), not just the average. The average always looks fine while individuals are drowning.

**Suggested gold table**
`fct_requisition` joined to `gold_dim_recruiter`, with weights from `gold_ref_role_complexity`.

**Executive rationale**
Answers *is the workload distributed fairly, and who is overloaded?* Overloaded recruiters are the leading indicator of both slipping time-to-fill and recruiter attrition — and replacing a recruiter costs the team a quarter of delivery.

---

### 12. Recruiter Capacity Gap

**Business definition**
The difference between the recruiting effort the current requisition load demands and the effort the team actually has available.

**Formula / calculation logic**
```
Demand Hours = SUM(Open Reqs × Estimated Hours per Req by complexity)
Available Hours = SUM(Recruiter FTE × Productive Hours per Period)

Capacity Gap = Demand Hours − Available Hours
Utilisation % = Demand Hours / Available Hours
```
- **Sign convention matters and must be locked down:** a *positive* gap means demand exceeds capacity (short-staffed). Document this on the page — this is the single most common source of misreading on a capacity tile.
- Productive hours should already net out admin time, leave, and non-req work. State the assumption openly.
- Add a simple scenario control (for example, "what if we add two recruiters" or "what if demand rises 15%") — capacity numbers are far more persuasive when leadership can test them.

**Suggested gold table**
`fct_recruiter_capacity` (recruiter × week: available hours, demand hours, weighted load) built from `fct_requisition` and `gold_dim_recruiter`.

**Executive rationale**
This is the headcount business case. It converts "the team feels stretched" into a defensible number, and it is the metric that justifies either hiring more recruiters or telling the business that the hiring plan is not deliverable with the current team.

---

### 13. Recruiter Throughput (Hires per Recruiter)

**Business definition**
How many hires each recruiter delivers per period, normalised for role complexity.

**Formula / calculation logic**
```
Throughput = Hires Delivered / Recruiter FTE in Period
Complexity-Adjusted Throughput = SUM(Hires × Complexity Weight) / Recruiter FTE
```
- Use a rolling 3-month window at executive level. Monthly recruiter throughput is too volatile to interpret.
- Present as a **team distribution**, not an individual ranking, on the executive page. Individual detail belongs in the recruiter scorecard, where it sits alongside quality measures.

**Suggested gold table**
`fct_hire` joined to `gold_dim_recruiter`, aggregated in `fct_recruiter_performance`.

**Executive rationale**
Throughput and capacity together answer *do we need more recruiters, or do we need to fix how the current team works?* A team at high utilisation with low throughput has a process problem; high utilisation with high throughput is a genuine staffing problem. Treat this metric with care — used alone it pushes teams to fill fast rather than fill well, which is exactly why it sits next to metrics 14 and 15.

---

## F. Hiring Outcomes

### 14. Mature-Cohort New Hire Attrition (60-Day)

**Business definition**
The percentage of new hires who leave within 60 days of starting — measured only on cohorts that have actually had 60 days to be observed.

**Measurement window: rolling 12 months of eligible cohorts, on a 60-day observation period**

There are two separate time settings here, and confusing them is the usual source of error:

| Setting | Value | What it controls |
|---|---|---|
| Observation period | 60 days from start date | How long each hire is watched for an exit |
| Cohort window | Rolling 12 months of start dates, ending 60 days before today | Which hires are included in the calculation |

```
Cohort window = start date BETWEEN (Today − 60 days − 365 days) AND (Today − 60 days)
```

**Why this window**
- **Rolling 12 months, not year-to-date.** Hiring is seasonal, and early attrition varies with the type of role being hired. A year-to-date figure in February is built on two months of hires and swings wildly; the same figure in November is stable. Rolling 12 months keeps the denominator a consistent size all year, so a movement in the metric is a real signal rather than an artefact of the calendar.
- **The window ends 60 days before today, not today.** Anyone who started in the last 60 days has not yet had the full observation period, so including them adds people to the denominator who mathematically *cannot* appear in the numerator. That silently pushes the rate down. Ending the window early is what makes the cohort "mature."
- **Why 60 days and not 90.** 60 days is early enough to isolate hiring and onboarding quality rather than longer-term management or role fit, and it returns a usable signal roughly a month sooner than a 90-day measure. If your business commonly uses 90-day probation, run both and headline the one that matches your probation period — but do not switch between them.
- **Trade-off to accept.** A rolling window smooths out a genuine sudden deterioration. If early attrition spikes in a single month, the rolling figure will move slowly. Pair the headline with a small trend line of monthly cohorts so a sharp change is still visible.

**Formula / calculation logic**
```
Eligible cohort = hires with start date within the cohort window above
60-Day Attrition % = Leavers within 60 days of start / Eligible cohort
```
- The **mature-cohort rule is essential**. Including recent hires who have not yet had time to leave mathematically suppresses the rate and makes the number look better than reality. This is the most common error in early-attrition reporting.
- Headline the mature cohort. Recent hires can be shown separately as "in observation" with their own count, so nobody thinks they were forgotten.
- Split voluntary vs involuntary — they point to different failures (hiring quality vs onboarding and role clarity).

**Suggested gold table**
`fct_new_hire_tenure` (hire × cohort with start date, exit date, exit type, and a precomputed `days_to_exit` plus a `cohort_matured_flag`), sourced from `fct_offer` and HR exit data.

**Executive rationale**
This is the counterweight to every speed metric on the dashboard. It answers *are we hiring the right people, or just hiring fast?* Early attrition is also expensive and highly visible to the business — a rising rate erodes TA's credibility faster than a missed time-to-fill target.

---

### 15. Hiring Manager Satisfaction

**Business definition**
How satisfied hiring managers are with the recruiting process and candidate quality, collected via a short survey at requisition close.

**Measurement window: rolling 90 days (surveys responded to in the last 90 days)**

```
Included = survey responses with response date BETWEEN (Today − 90 days) AND Today
```

**Why this window**
- **Rolling 90 days, not quarter-to-date.** A quarter-to-date figure resets to near-zero responses every three months, so the number is unusable in the first weeks of each quarter and only becomes trustworthy near the end. A rolling window carries a stable response base at every point in time.
- **90 days rather than 30.** Survey response volume is low — most organisations close a modest number of requisitions per month, and only a fraction of managers respond. A 30-day window often produces a score built on a handful of responses, where one unhappy manager swings the number several points. 90 days usually gets the base to a size where movement means something.
- **Do not go beyond 90 days.** A rolling 6- or 12-month window is so slow to move that a real deterioration in service takes half a year to surface, which defeats the purpose of an early-warning metric.
- **Anchor on response date, not requisition close date.** Managers often respond weeks after the req closes. Anchoring on close date makes the recent end of the window look artificially thin as late responses trickle in and quietly restate the figure.
- **Set a minimum response threshold.** Below roughly 20 responses in the window, suppress the score for that slice and show the response count instead. Publishing a satisfaction score from five responses invites decisions the data cannot support — especially when it is sliced by business unit or recruiter, where volumes are much smaller than the overall figure.

**Formula / calculation logic**
```
HM Satisfaction = MEAN(survey score) or NPS = %Promoters − %Detractors
Response Rate = Responses in window / Surveys Sent in window
```
- Always publish **response rate** alongside the score. A 9.1 from 12% of managers is not a fact, it is a rumour.
- Report the score as a trend of rolling values, not a single point — direction matters more than level, since absolute satisfaction scores are hard to benchmark meaningfully.
- Segment by business unit and recruiter to make it actionable, subject to the minimum response threshold above.

**Suggested gold table**
`fct_survey_response` (survey × requisition × hiring manager, with score, response date, requisition close date, sentiment, and free-text theme tags).

**Executive rationale**
TA's internal customer is the hiring manager. Every other metric on this dashboard can look healthy while the business quietly loses confidence in the function. This is the metric that catches that early — and it is usually the first thing a new CHRO asks about.

---

## Summary Table — For Semantic Model Translation

| # | Metric | Business Area | Core Calculation | Suggested Gold Table | Grain | Executive Question Answered |
|---|--------|---------------|------------------|----------------------|-------|------------------------------|
| 1 | Offer Plan Attainment (To Date) | Plan & Delivery | Offers accepted to date ÷ planned offers to date | `fct_offer` + `fct_hiring_plan` | BU × job family × month | Are we delivering the plan? |
| 2 | Forecasted Attainment (Next End of Month) | Plan & Delivery | (Offers accepted this month + probable accepts within days remaining) ÷ current-month plan | `fct_requisition` + `fct_offer` + `fct_hiring_plan` | BU × job family × month | Are we on pace to hit this month? |
| 3 | Offers at Risk | Plan & Delivery | Count of open reqs with projected offer-accept date > target hire date | `fct_requisition` | Requisition | What is about to slip? |
| 4 | Median Time to Fill (mix-adjusted) | Operations | MEDIAN(precomputed `time_to_fill_days`) on accepted offers, weighted to baseline role mix | `fct_offer` | Offer (accepted) | How fast are we, genuinely? |
| 5 | Median Time in Stage | Operations | Median(stage exit − stage entry) by stage | `fct_application_stage` | Candidate × stage | Where is the bottleneck? |
| 6 | Pipeline Coverage Ratio | Funnel | Active late-stage candidates ÷ open reqs | `fct_application_stage` + `fct_requisition` | Job family × BU | Can we meet future demand? |
| 7 | Stage Conversion Rate | Funnel | Cohort entering next stage ÷ cohort entering current stage | `fct_funnel_conversion` | Stage × cohort | Why is the funnel leaking? |
| 8 | Source Effectiveness | Funnel | Hires from source ÷ candidates from source | `fct_source_performance` | Source × period | Where should sourcing spend go? |
| 9 | Offer Acceptance Rate | Offer | Offers accepted ÷ offers extended (cohort) | `fct_offer` | Offer | Are we winning candidates? |
| 10 | Offer Renege & Rescind Rate | Offer | Accepted offers that never started ÷ accepted offers | `fct_offer` + `fct_hire` | Offer | Why do accepts not become starts? |
| 11 | Recruiter Requisition Load | Capacity | Σ(open reqs × complexity weight) per recruiter | `fct_requisition` + `gold_dim_recruiter` | Recruiter | Who is overloaded? |
| 12 | Recruiter Capacity Gap | Capacity | Demand hours − available hours (positive = short-staffed) | `fct_recruiter_capacity` | Recruiter × week | Do we need more recruiters? |
| 13 | Recruiter Throughput | Capacity | Complexity-weighted hires ÷ recruiter FTE (rolling 3 months) | `fct_recruiter_performance` | Recruiter × period | Is it a staffing or process problem? |
| 14 | Mature-Cohort 60-Day Attrition | Outcomes | Leavers within 60 days ÷ eligible (matured) cohort — **rolling 12-month cohort window ending 60 days before today** | `fct_new_hire_tenure` | Hire cohort | Are we hiring well, not just fast? |
| 15 | Hiring Manager Satisfaction | Outcomes | Mean score or NPS, with response rate — **rolling 90 days by response date, min. 20 responses** | `fct_survey_response` | Requisition × manager | Does the business trust TA? |

---

## Overall Rationale — Why This Set Works Together

These fifteen metrics are not a list of everything TA can measure. They are a chain of reasoning that walks an executive from symptom to cause to action in a single page sequence.

**Are we delivering against the hiring plan?**
Metrics 1 and 2 answer this directly, and answer it in the language the business already uses — plan versus actual, with a forecast. Metric 10 protects the answer's honesty by explaining the gap between accepted offers and actual starts.

**Are we likely to meet future hiring demand?**
Metric 6 (pipeline coverage) is the early warning system, supported by metric 2's forecast and metric 8 (source effectiveness), which shows whether the channels feeding the pipeline can be scaled. Coverage moves months before delivery does, which is precisely what makes it valuable.

**Where are the recruitment bottlenecks?**
Metric 4 tells you speed is a problem; metric 5 tells you *where*, including whether the delay sits with the hiring manager rather than the recruiter. Metric 7 tells you whether the problem is volume moving through the funnel or quality entering it. Together they turn one aggregate number into a specific intervention.

**Are recruiters operating within sustainable capacity?**
Metrics 11, 12, and 13 form a set that must be read together. Load shows distribution, gap shows the size of the shortfall, throughput shows whether extra people would actually help. Any one of them alone leads to a wrong conclusion — high load with high throughput means hire more recruiters; high load with low throughput means fix the process first.

**Are candidates progressing and accepting offers effectively?**
Metric 7 covers progression, metric 9 covers acceptance, and metric 10 covers whether acceptance holds. Declining acceptance usually signals a compensation or speed issue, which links back through metrics 4 and 5 — the fastest way to improve acceptance is often to shorten the process, not to raise offers.

**Where should TA leadership intervene?**
This is the point of the whole design. Metric 3 (offers at risk) produces the specific list of requisitions needing attention; metrics 5, 6, 8, and 12 supply the four standard interventions — unblock a stage, add pipeline, redirect sourcing spend, or rebalance recruiter load. The dashboard should not merely display these numbers; it should surface the requisitions each one implicates.

**The deliberate balance**
Metrics 1–13 measure delivery, speed, and volume. Left alone, they create pressure to fill roles fast and cheap. Metrics 14 and 15 are the counterweight — early attrition catches poor hiring quality, and hiring manager satisfaction catches a process that is efficient on paper but painful for the business. Without them, this dashboard would quietly optimise the function in the wrong direction.

---

## What Was Deliberately Excluded, and Why

| Excluded metric | Reason |
|---|---|
| Cost per hire | Backward-looking, heavily influenced by role mix, and rarely changes an executive decision at weekly or monthly cadence. Belongs in an annual budget review. |
| Application volume | High volume signals reach, not quality. Pipeline coverage (metric 6) captures what actually matters. |
| Interviews scheduled / conducted | Activity, not outcome. Useful for recruiter coaching, not executive decisions. |
| Candidate NPS | Genuinely valuable, but response rates are usually too low and too biased for an executive number. Keep it as a supporting drill-through. |
| Diversity representation | Important, but it needs its own governed page with careful definitions, legal review, and jurisdiction-specific handling — not a tile on a delivery dashboard. Treat it as a companion product, not an omission. |
| Requisition aging | Largely redundant with offers at risk (metric 3), which is more predictive. Keep as a drill-through detail. |

---

## Implementation Notes for the Semantic Model

1. **Define the fact grain before the measure.** Requisition-grain, offer-grain, application-stage-grain, and recruiter-week-grain are four distinct fact tables. Attempting to serve all fifteen metrics from one wide table will produce filter-context problems in DAX that are painful to unwind later.
2. **Lock the sign conventions and denominators in a published metric dictionary** — particularly capacity gap direction (metric 12), offer acceptance denominator (metric 9), and the mature-cohort rule (metric 14). These three are where executive dashboards most often lose credibility.
3. **Make restatement visible, not hidden.** Reneges and rescinds change historical counts. An "as of" stamp and a restatement note on the page is far cheaper than rebuilding trust after someone spots the change.
4. **Cohort-based measures need a bridge or calculation group**, since they filter on the cohort's entry date rather than the reporting date. Plan for this early — retrofitting cohort logic into a date-filtered model is expensive.
5. **Keep benchmark and weighting tables governed** (`gold_ref_role_complexity`, `gold_ref_stage_benchmark`, `gold_ref_role_mix_baseline`). These are business assumptions, not code, and they should be reviewable and versioned by the TA leadership team rather than buried in a measure.
