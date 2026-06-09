# Feature Catalog — Regime-Aware Churn Predictor

Candidate features for the 30-day churn model, organized by source CSV.

**Ground rules**
- All features are **point-in-time**: computed looking back from each member's `as_of_date`, using **only data strictly before** that date (no leakage). Engagement events run past the holdout `as_of_date` (to 2024-11-19) — they must be cut off.
- One **`build_features(member_ids, as_of_date)`** function used identically for **train** (`as_of_date` 2024-07-01) and **holdout** (`as_of_date` 2024-10-01).
- Sparse sources need **missing-value flags**, not silent zeros (e.g. "win rate with no signals" ≠ 0).
- Prefer **continuous market features + interactions** over hard bull/bear/recovery buckets — the holdout regime may be out-of-distribution.

---

## 1. `members.csv` — static profile (cheap, always present)

| Feature | What it captures | Why it predicts churn |
|---|---|---|
| `tenure_days` | days from signup → as_of_date | new members churn more |
| `acquisition_channel` (one-hot) | how they came in | paid/exchange cohorts churn differently |
| `plan_variant` | basic vs premium vs quarterly | quarterly = pre-committed, stickier |
| `had_trial` | trialed before paying | trial converts often churn faster |
| `region`, `age_bracket`, `signup_device` | demographics | cohort effects (read region with `keep_default_na=False` — `NA` = North America, not null) |
| `initial_discord_join` | joined community early | early embedding = retention |
| `signup_btc_price` vs price at as_of | bought high vs low | underwater members churn |

## 2. `engagement_events.csv` — the core churn signal

| Feature | What it captures | Why |
|---|---|---|
| `days_since_last_event` | **recency** | the #1 churn predictor |
| `events_7d / 30d / 90d` | **frequency** by window | activity level |
| `events_by_type_30d` (5 counts) | stream / signal / discord / login / acted | which behavior |
| `engagement_trend` (last 30d ÷ prior 30d) | **ramping up or down** | decline → churn; **regime-sensitive** |
| `active_days_30d` | distinct active days | consistency vs binge |
| `avg_session_duration` | depth of use | shallow = disengaging |
| `discord_msgs_30d` | community participation | social tie = sticky |

## 3. `signals_taken.csv` — did the product deliver? (sparse: 84/108 holdout)

| Feature | What it captures | Why |
|---|---|---|
| `signals_taken_90d` | usage of core value | engaged with product |
| `signal_win_rate` | % profitable | losing money → churn |
| `cum_pnl_usd / avg_pnl_pct` | money made/lost | the value proposition |
| `days_since_last_signal` | recency of acting | drop-off |
| `has_taken_signal` (flag) | ever acted | + missing flag |

## 4. `support_tickets.csv` — frustration (sparse: 68/108 holdout)

| Feature | What it captures | Why |
|---|---|---|
| `tickets_90d` | contact volume | friction |
| `has_trust_ticket` / `has_billing_ticket` | category flags | **trust = strong churn smell** |
| `open_tickets` (unresolved) | unresolved pain | live frustration |
| `days_since_last_ticket` | recency | recent issue |

## 5. `market_context.csv` — regime (the differentiator)

| Feature | What it captures | Why |
|---|---|---|
| `btc_return_30d / 90d` (trailing) | bull vs bear | flips feature meaning |
| `btc_realized_vol_30d` | turbulence | capitulation driver |
| `fear_greed_index` | sentiment | 25 = fear vs 64 = greed |
| `funding_rate` | leverage froth | late-cycle risk |
| `regime_x_engagement` (interactions) | engagement × vol | **lets model learn the inversion** |

## Cross-cutting

- **Missing flags** for every sparse source: `no_signals`, `no_tickets`, `no_events`.
- **Interactions** (e.g. `engagement_trend × vol`) capture "heavy engagers churn first in bear" without hard regime buckets.

---

## Priority

High-value core: **engagement recency / frequency / trend + market regime + interaction terms**.
Signals & tickets add lift but are sparse → lean on flags. Profile is the cheap baseline.
