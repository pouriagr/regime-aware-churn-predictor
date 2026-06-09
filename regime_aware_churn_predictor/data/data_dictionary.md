# Data dictionary

All files in `/data/`. CSV with header rows. Empty string means null.

## members.csv
One row per Banter Pro member.

| column | type | notes |
|---|---|---|
| member_id | string | primary key, format `M{int}` |
| signup_date | date (YYYY-MM-DD) | UTC |
| acquisition_channel | enum | youtube_organic, twitter_paid, podcast, referral, exchange_partnership, other |
| region | enum | NA, EU, LATAM, MENA, APAC, OTHER |
| signup_btc_price | float | BTC close on signup_date |
| plan_variant | enum | monthly_basic, monthly_premium, quarterly_premium |
| had_trial | int (0/1) | took a free trial before paying |
| age_bracket | enum | 18-24, 25-34, 35-44, 45-54, 55+ |
| preferred_language | string | ISO 639-1 |
| payment_method | enum | stripe_card, crypto_usdc, crypto_btc, paypal |
| referrer_member_id | string | nullable; populated for referral channel |
| utm_source | string | nullable |
| utm_campaign | string | nullable |
| initial_discord_join | int (0/1) | joined discord within 7d of signup |
| opted_in_newsletter | int (0/1) | |
| signup_device | enum | mobile, desktop, tablet |
| lifetime_signal_count_at_signup | int | total signals available at signup; populated from internal counter |

## engagement_events.csv
~tens of thousands of rows. One row per event.

| column | type | notes |
|---|---|---|
| event_id | string | |
| member_id | string | FK to members.csv |
| timestamp_utc | datetime | as logged by source service |
| event_type | enum | stream_view, signal_view, signal_acted, discord_message, dashboard_login |
| session_duration_sec | int | nullable; only for stream_view and dashboard_login |
| content_id | string | nullable; stream/signal identifier |
| source_service | enum | primary, legacy_v1 |

## signals_taken.csv
Member self-reported acting on a Banter signal.

| column | type | notes |
|---|---|---|
| member_id | string | FK |
| signal_id | string | |
| taken_at | datetime | when the member reported taking the signal |
| direction | enum | long, short |
| size_usd | int | self-reported position size |
| realized_pnl_pct | float | percent return realized on the signal |
| realized_pnl_usd | float | size_usd × realized_pnl_pct |

## support_tickets.csv

| column | type | notes |
|---|---|---|
| ticket_id | string | |
| member_id | string | FK |
| opened_at | datetime | |
| category | enum | billing, access, content, trust, other |
| subject | string | free text |
| body_snippet | string | free text, truncated |
| resolved_at | datetime | nullable; null if still open |

## market_context.csv
Daily, UTC.

| column | type | notes |
|---|---|---|
| date | date (YYYY-MM-DD) | UTC day, daily aggregations close at 00:00 UTC |
| btc_open | float | |
| btc_high | float | |
| btc_low | float | |
| btc_close | float | |
| btc_volume_usd | float | |
| btc_realized_vol_30d | float | annualized |
| funding_rate_avg_8h | float | volume-weighted across major perp venues |
| fear_greed_index | int | 0-100 |
| total_crypto_mcap_usd | float | |

## labels.csv
Labelled window only.

| column | type | notes |
|---|---|---|
| member_id | string | FK |
| as_of_date | date | snapshot date the label is computed from |
| churn_date | date | nullable; date member churned, if within 30d of as_of_date |

## holdout_scoring.csv

| column | type | notes |
|---|---|---|
| member_id | string | FK |
| as_of_date | date | snapshot date for which to predict |

Submit predictions.csv with: `member_id,churn_probability`. One row per member_id in holdout_scoring.csv. Probabilities in [0,1].
