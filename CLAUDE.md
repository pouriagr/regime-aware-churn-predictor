IN CHATS ANSWER THE QUESTIONS SHORT, TO THE POINT AND EASY TO READ FOR A NON-NATIVE ENGLISH SPEAKER.

# Regime-Aware Churn Predictor

Take-home assessment (Senior AI/ML Engineer). Full brief: `regime_aware_churn_predictor/docs/Assessment_-_Senior_AI_ML_Engineer.pdf`. Data: `regime_aware_churn_predictor/data/`.

> Scope note: designed as a ~90-min exercise. They judge **approach and reasoning**, not perfection. Don't over-engineer.

## The business problem

Crypto Banter runs a paid tier ("Banter Pro", ~38k members, monthly billing, cancel anytime). The retention team wants to fire a **win-back offer** (30-day discount + 1-on-1 onboarding call) at members an ML model predicts will **churn in the next 30 days**.

**Goal:** build a model that ranks members by 30-day churn probability, recommend an operating point (whom to target), and write a deployment plan that survives the next regime shift.

## Why it's hard: regime inversion

22 months of data span a full **bull → bear → recovery** cycle. Feature meanings *invert* across regimes:

- **Bull:** signups flood in, churn low (~3%/mo), members forgiving.
- **Bear:** mass churn (~14%/mo) and **patterns flip** — heavy-engaging members churn first (check obsessively, then capitulate); low-engagement members ride it out.
- **Recovery:** new cohort, different acquisition-channel mix.

A bull-trained model deployed in bear *inverts* (worse than useless). A regime-blind model predicts the useless average. **The model is deployed in an unknown future regime.**

### How to handle regime without a regime label
The holdout gives no regime name — but each row has an **`as_of_date`**. Join that date to `market_context.csv` to derive **continuous** market features (volatility, trend, drawdown, fear/greed). Build them identically on train and holdout. Prefer continuous market signals over hard bull/bear/recovery buckets, because the holdout regime may sit **outside the range seen in training**.

## The economics (drives the operating point)

| Lever | Value |
|---|---|
| Intervention cost (margin + community-lead time) | **$28** per member |
| Save rate (would-be churners retained by offer) | **22%** |
| Cost of an unprevented churn (LTV) | **$190** |
| Budget cap | offer to **≤ 6%** of active base / month |

Per targeted real churner: 22% → +$190 (saved), 78% → wasted $28 (lost cause). So even a perfect churn model wastes money on the un-saveable majority. **Note the uplift gap in the write-up:** ranking by churn probability ≠ ranking by *saveability*; ideal target = likely-to-churn **AND** likely-to-respond. No experiment data exists to model uplift here, so churn probability is the accepted proxy — but call out that a proper solution would A/B test offers and model uplift.

## Required deliverables

1. **`predictions.csv`** — exact format `member_id,churn_probability`, one row per member in `holdout_scoring.csv`, probs in [0,1]. **Calibration is scored, not just ranking** (if you say 0.2, ~20% should actually churn).
2. **Recommended operating point** — the cutoff for who gets the offer, respecting the ≤6% budget and the cost math. They compute **realized net margin** at this point on the hidden holdout.
3. **Deployment plan** (written) — monitoring, recalibration, regime-shift safeguards, fallback.

## Data (`regime_aware_churn_predictor/data/`)

Join everything by **`member_id`**; join to market by **`date`** (= `as_of_date`). Schema in `data/data_dictionary.md`.

| File | Contents | Use |
|---|---|---|
| `members.csv` | Static profile: signup_date, acquisition_channel, region, signup_btc_price, plan_variant, had_trial, age_bracket, payment_method, referrer_member_id, device, discord/newsletter flags, etc. | Demographic / acquisition features |
| `engagement_events.csv` | One row per event (~tens of thousands, despite "41M" in the story). event_type ∈ {stream_view, signal_view, signal_acted, discord_message, dashboard_login}, timestamp, session_duration_sec. | Richest churn signal — build recency/frequency/trend features |
| `signals_taken.csv` | Self-reported signal actions + **realized P&L** (pct & usd), direction, size. | "Did signals make them money?" features |
| `support_tickets.csv` | Free text: category (billing/access/content/**trust**/other), subject, body_snippet, opened/resolved times. | Frustration signal |
| `market_context.csv` | Daily BTC OHLCV, btc_realized_vol_30d, funding_rate_avg_8h, fear_greed_index, total_crypto_mcap_usd. | **Regime features — join on date** |
| `labels.csv` | `member_id, as_of_date, churn_date` (churn within 30d of snapshot). | Training labels |
| `holdout_scoring.csv` | `member_id, as_of_date` only — no labels. | The members to predict |

## Critical gotchas

- **No leakage:** every feature for a member must use only data **strictly before** that member's `as_of_date`. The 30-day label window looks forward from `as_of_date`.
- **Calibration matters** — fit/check it (e.g. reliability curve, isotonic/Platt) since the holdout regime base rate differs from training.
- **Messy data on purpose:** nulls (empty string = null), noise, production quirks. Expect cleaning.
- **Region `NA` quirk:** region `NA` = North America (2nd-largest region, 77 members), but `pandas.read_csv` coerces the string `"NA"` to NaN by default. The dictionary says *only empty string = null*, so read every file with `keep_default_na=False, na_values=['']`. Otherwise North America silently vanishes into "missing". (Handled in the feature notebook §1.2.)
- **Per-member snapshots:** a member may appear at a given `as_of_date`; features are point-in-time as of that date.
- **Holdout regime is unknown and possibly out-of-distribution** — favor robust, continuous, regime-aware features over regime-specific tricks.
