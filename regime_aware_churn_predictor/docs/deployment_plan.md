# Deployment Plan — Regime-Aware Churn Predictor

## What we deploy
- **Model:** L2 logistic regression (`C=0.05`), features scaled in-pipeline, refit on all labeled data. Linear → it **extrapolates the regime interactions** (`vol × engagement`) that trees can't, and calibrates well on small data.
- **Output:** monthly calibrated 30-day churn probability per member, point-in-time (only data **before** `as_of_date` — no leakage).
- **Honest performance:** ROC ≈ 0.60, Brier ≈ 0.126 (beats flat-rate 0.128). Modest — regime flip + 18 training positives.

## Operating point (whom to target)
- Break-even precision = `28 / (0.22 × 190)` ≈ **0.67**. We reach only ~0.30 → **net margin is negative at every point.**
- **Recommend: launch narrow** — target only the **top ~2%** (near break-even) as a **monitored pilot**; do **not** run the full 6% as priced.

## Monitoring (monthly)
- **Regime features** (BTC return/vol/fear-greed) drift **outside training range** → predictions are extrapolations, lower trust.
- **Score distribution** mean/spread shift → population drift.
- **Realized churn vs predicted** → calibration drift (triggers recalibration).
- **Precision@target & net margin** → if net margin negative 2 months → **pause**.
- **Feature health** — null rates, new `region`/`channel` categories.

## Recalibration
- Use the built-in **prior-shift**: `logit_new = logit_old + [logit(π_target) − logit(π_train)]`. Monotonic → **never changes who is targeted**, only fixes probability values.
- **Trigger:** once realized churn is observed, set `π_target` = realized rate. Re-fit the model quarterly / per new labeled cohort.

## Regime-shift safeguards
- **Continuous** market features (no hard bull/bear buckets) → out-of-range regimes still score sensibly.
- **Out-of-range guard:** flag predictions as "extrapolated / low-confidence" and add human review.
- **Champion/challenger:** keep the flat-base-rate baseline live; fall back if the model stops beating it on Brier.

## Fallback
- Model < baseline (2 months) → **rules** (target members with recent billing/trust tickets + engagement decline).
- Inputs out-of-distribution → human-reviewed rules, no auto-fire.
- Net margin negative → **pause campaign** (the offer economics, not the model, are the bottleneck).

## Key limitation → next step
**Churn ≠ saveability.** We rank by churn, but the offer only converts 22%. Run the pilot as an **A/B test** (random no-offer holdout) → model **uplift** and retarget by *likely-to-churn AND likely-to-respond*. That, plus better offer economics, is the path to positive margin.
