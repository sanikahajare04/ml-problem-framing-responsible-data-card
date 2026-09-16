# ML Problem-Framing Memo: Subscription Churn Retention

**Status:** Draft for review
**Author:** [name] | **Date:** [date] | **Reviewers:** [product, legal/privacy, eng lead]

---

## 1. The decision this model informs

**Business decision:** Which active subscribers should receive a proactive retention
intervention (discount offer, support outreach, or win-back call) this billing cycle?

This is *not* "will this customer churn" in the abstract — it is a decision about
**where to spend a limited retention budget/agent-hours** before the customer leaves.
Framing it as a decision, not a prediction, is what lets us pick a non-ML baseline and
compute real costs below.

## 2. Prediction target

- **Target variable:** `churned` — binary, 1 if the customer cancels/lapses, 0 otherwise.
- **Label definition (must be nailed down before training, not left implicit):**
  - What counts as "churn" — voluntary cancellation only, or also non-renewal,
    downgrade, or payment failure? The provided sample only has a flat `churned`
    flag with no definition attached — **this is a gap to close with the data owner
    before this is usable** (see Data Card, Section 3).
  - Over what horizon is churn measured (e.g., churns within the next 30 days)?
- **Unit of observation:** one row = one customer, evaluated at a snapshot in time
  (not one row per event/login).
- **Action window:** predictions must be generated with enough lead time for an
  intervention to matter — proposed **T-30 days**: score customers 30 days before
  their renewal date so retention outreach has time to work. A model that only
  flags churn the day before renewal is not actionable.

## 3. Non-ML baseline (must beat this to justify ML)

Before committing to an ML system, we are obligated to check whether a simple,
auditable rule gets us most of the way there. Two baselines were tested against the
12-row sample dataset:

| Baseline | Rule | Accuracy | Precision | Recall | F1 |
|---|---|---|---|---|---|
| A — Majority class | Always predict "not churned" | 0.58 | 0.00 | 0.00 | 0.00 |
| B — Simple business rule | Flag if `last_login_days > 10` **OR** `support_tickets >= 4` | **1.00** | **1.00** | **1.00** | **1.00** |
| ML — Logistic Regression (leave-one-out CV) | tenure, tickets, spend, last-login, plan type | **1.00** | **1.00** | **1.00** | **1.00** |

**Finding:** on this small sample, the two-line business rule (Baseline B) performs
identically to the logistic regression model. This is expected — 12 rows is not
enough to distinguish a genuine ML advantage from overfitting/coincidence, and the
sample appears to have been constructed so churn is cleanly separable. **The
practical implication is important: this is exactly the situation where a team
should be suspicious of "we need ML" and should re-run this same baseline
comparison on the full, real dataset before approving model development.** If the
rule keeps performing this well at scale, ML is not justified for this decision —
the rule is cheaper to build, explain to support staff, audit, and defend to a
customer who asks "why was I flagged?"

**Decision gate:** ML work should only proceed if, on a real (≥ a few thousand row)
holdout set, the ML model beats Baseline B by a business-metric margin large enough
to offset its added cost in complexity, explainability, and monitoring (see Section
5). This memo does not yet claim that gate is met.

## 4. Business and model metrics, with explicit costs

| Outcome | What it means operationally | Estimated cost |
|---|---|---|
| **False negative** (miss a churner) | Customer leaves with no intervention attempt | Lost customer lifetime value (LTV). For this product, avg. Pro/Standard LTV ≈ ₹15,000–25,000; Basic ≈ ₹3,000–5,000. **This is the expensive error.** |
| **False positive** (flag a loyal customer) | Retention team spends an outreach slot / offers an unneeded discount | Cost of the intervention itself (agent time + discount value), roughly ₹200–500, plus a small trust cost if the outreach feels intrusive |
| **True positive** | Correctly flagged, intervention may save the customer | Intervention cost, offset by probability of retention |
| **True negative** | Correctly left alone | No cost |

Because false negatives are far more expensive than false positives here, the
**business metric is recall-weighted**, not plain accuracy:
- **Primary business metric:** expected net savings = (TP × retained-value ×
  save-rate) − (TP+FP) × intervention-cost − FN × LTV. Model selection should
  optimize this, not accuracy or F1 alone.
- **Primary model metric:** **recall at a fixed, budget-constrained precision**
  (e.g., "maximize recall while keeping precision ≥ 40%," since the retention team
  can only handle a fixed number of outreach attempts per cycle).
- **Secondary model metrics:** PR-AUC (more informative than ROC-AUC given class
  imbalance), calibration of predicted probabilities (since the output feeds a
  ranked worklist, not just a binary flag).
- **Guardrail metric:** false-positive rate within any plan tier or demographic
  group should not diverge sharply (see fairness note in Data Card).

## 5. Abstention, human review, monitoring, rollback

- **Abstention band:** customers with predicted churn probability in a mid-range
  band (e.g., 0.35–0.55) are routed to a human reviewer rather than auto-scored,
  since this is where the model is least confident and the cost of a wrong
  automated call is highest.
- **Human-in-the-loop:** the model produces a ranked worklist and a probability,
  not an automatic action. A retention agent decides the actual intervention. No
  customer is auto-charged, auto-discounted, or auto-contacted without a person in
  the loop for the initial launch.
- **Monitoring:**
  - Weekly: prediction volume, score distribution drift, precision/recall against
    realized 30-day outcomes.
  - Monthly: subgroup performance by `plan_type` and tenure band, to catch
    disparate error rates early.
  - Alert if the live business metric (Section 4) falls below the non-ML
    baseline's performance for two consecutive cycles.
- **Rollback conditions:** revert to Baseline B (the business rule) if any of:
  1. Realized precision drops more than 15 points below backtest precision for
     two consecutive scoring cycles.
  2. A subgroup shows a false-negative rate more than 1.5× the overall rate.
  3. Data pipeline changes upstream (e.g., a support-ticket system migration)
     change feature semantics without a re-validated model.
  4. Legal/privacy flags an issue with a data source (see Data Card, Section 6).

## 6. Open questions before training on real data

1. Who owns the definition of "churned," and is it consistent across the historical
   data used for training?
2. What is the actual retention-team capacity per cycle (this sets the operating
   threshold, not F1)?
3. Is `plan_type` or any correlated feature a legally protected or sensitive
   attribute in this market? (See Data Card.)
4. Do we have consent/purpose-limitation coverage to use support-ticket content and
   login timestamps for this purpose?
