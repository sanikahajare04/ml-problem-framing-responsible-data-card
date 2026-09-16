# Risk Register — Churn Retention Model

| ID | Risk | Category | Likelihood | Impact | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|---|
| R1 | Label `churned` is inconsistently defined/applied across the historical data | Data quality | Medium | High | Confirm label definition and cutoff logic with data owner before training on real data; document in Data Card §3 | Data eng lead | Open |
| R2 | Temporal leakage: features computed after the churn decision (e.g., cancellation-related support tickets) | Leakage | Medium | High | Enforce a hard feature cutoff date (T-30) in the ETL; add an automated leakage check comparing feature timestamps to label timestamp | ML lead | Open |
| R3 | Simple business rule (Baseline B) matches ML performance on available data — ML may not be justified | Problem framing | High (on sample) | Medium | Re-run baseline comparison on full real dataset before approving ML build-out; require a documented margin over baseline to proceed | Product + ML lead | Open — gate defined in memo §3 |
| R4 | `plan_type`/`monthly_spend_inr` act as proxies for socioeconomic status, risking disparate false-negative rates by group | Fairness | Medium | High | Run subgroup precision/recall checks pre-launch and monthly post-launch; alert if any subgroup FNR > 1.5× overall | ML lead | Open |
| R5 | No documented consent basis for using login/support behavioral data for retention scoring | Privacy/consent | Medium | High | Legal/privacy review of ToS and consent basis before training on real customer data; exclude any opted-out customers | Legal/privacy | Open |
| R6 | Small-sample metrics (12 rows) misread as evidence of production-ready accuracy | Evaluation validity | High | Medium | Explicitly label all current metrics as illustrative in memo and notebook; require a real holdout evaluation before launch claims | ML lead | Mitigated in documentation |
| R7 | Model output used for an out-of-scope decision (e.g., pricing or credit) by another team | Scope creep / misuse | Low | High | Document out-of-scope uses in Data Card §1; restrict access to the scoring output to the retention system only | Product owner | Open |
| R8 | Automated action taken directly on model score without human review | Governance | Low | High | Enforce human-in-the-loop design (memo §5); no auto-discount/auto-cancel actions in v1 | Eng lead | Mitigated by design |
| R9 | Missingness in production data (e.g., customers who never logged in) handled inconsistently or silently imputed | Data quality | Medium | Medium | Define explicit missingness policy per feature before training (Data Card §6) | Data eng lead | Open |
| R10 | Model/pipeline drift after upstream system changes (e.g., support platform migration) silently degrades features | Monitoring | Medium | Medium | Weekly drift monitoring on feature distributions; rollback trigger defined in memo §5 | ML lead | Open |

**Review cadence:** revisit this register at each of (a) real-data baseline
re-run, (b) pre-launch review, (c) monthly post-launch monitoring review.
