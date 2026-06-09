# Regime-Aware Churn Predictor

Rank Crypto Banter "Banter Pro" members by **30-day churn probability**, recommend whom to target with a win-back offer, and write a deployment plan that survives the next market-regime shift. **The hard part:** the model is trained on a **bear** snapshot but scored on an unseen **recovery** regime, where churn drivers invert.

---

## How to run

```bash
poetry install                       # installs pandas, scikit-learn, jupyterlab, etc.

# Option A — interactive.
poetry run jupyter lab
#   then run, in order:
#   1) regime_aware_churn_predictor/notebooks/feature_engineering.ipynb
#   2) regime_aware_churn_predictor/notebooks/modeling.ipynb

# Option B — headless (re-runs everything, regenerates predictions.csv)
poetry run jupyter nbconvert --to notebook --execute --inplace \
  regime_aware_churn_predictor/notebooks/feature_engineering.ipynb
poetry run jupyter nbconvert --to notebook --execute --inplace \
  regime_aware_churn_predictor/notebooks/modeling.ipynb
```

**Run order matters:** `feature_engineering.ipynb` writes the feature tables that `modeling.ipynb` reads. Notebooks auto-locate the data, so they run from anywhere.

> Raw challenge CSVs and the brief PDF are **gitignored** (not redistributable). Place them under `regime_aware_churn_predictor/data/` and `docs/` to reproduce.

---

## Where each requested result lives

**The 3 required deliverables:**

| # | Deliverable | Location |
|---|---|---|
| 1 | **`predictions.csv`** — `member_id,churn_probability`, 108 rows, calibrated | **`regime_aware_churn_predictor/predictions.csv`** (written by `modeling.ipynb` §5) |
| 2 | **Recommended operating point + net-margin** | `modeling.ipynb` **§6** |
| 3 | **Deployment plan** (monitoring, recalibration, safeguards, fallback) | **`docs/deployment_plan.md`** |

**Supporting analysis:**

| Question | Location |
|---|---|
| Data understanding / regime evidence | `feature_engineering.ipynb` **Part 1 (EDA)** — §1.5 regime plot |
| Feature design & no-leakage logic | `feature_engineering.ipynb` **Part 2**; rationale in **`docs/features.md`** |
| Model comparison & selection | `modeling.ipynb` §2.2 (reg sweep), §2.3 (candidates) |
| ROC curve (best model) | `modeling.ipynb` §2.4 |
| Feature importance (global drivers) | `modeling.ipynb` §2.5 |
| Precision/recall vs threshold | `modeling.ipynb` §2.6 |
| Calibration check (reliability curve) | `modeling.ipynb` §2.7 |
| Per-member predictions **with reasons** | `modeling.ipynb` §3 → `data/processed/holdout_predictions_explained.csv` |
| Regime recalibration (prior-shift) | `modeling.ipynb` §4 |

---

## Key decisions & results

- **Regime via continuous market features**, not hard bull/bear buckets — joined by `as_of_date`, built identically on train/holdout, so an out-of-range holdout still scores sensibly. The regime signal rides in **interaction terms** (`vol × engagement`), which a **linear** model can extrapolate but trees cannot.
- **Model:** strongly-regularized **L2 logistic regression** (`C=0.05`) beat trees / elastic-net / KNN on calibration *and* ranking; selected by **precision@6%** with a calibration guard (must beat the flat-base-rate Brier). All candidates tie within CV noise — the binding constraint is **data, not model**.
- **Honest performance:** ROC ≈ 0.60, Brier ≈ 0.126 (beats flat-rate 0.128). Modest, because training is bear / only 18 churners / unseen regime.
- **Calibration:** verified on training (reliability curve, §2.7); the holdout is a different base rate → handled by a **prior-shift** recalibration hook (monotonic → never changes who is targeted).
- **Economics:** break-even precision ≈ **0.67** vs our ~0.30 → **net margin is negative at the 6% budget.** Recommend launching **narrow (top ~2%) as a monitored A/B pilot**, and — the core insight — model **uplift** (churn ≠ saveability) rather than running the blanket offer as priced.

---

## Repo layout

```
regime_aware_churn_predictor/
├── data/                          # raw challenge CSVs (gitignored) + data_dictionary.md
│   └── processed/                 # train/holdout feature tables + explained predictions
├── docs/
│   ├── Assessment_..._.pdf        # the brief (gitignored)
│   ├── features.md                # feature catalog (per-source, with rationale)
│   └── deployment_plan.md         # deliverable 3
├── notebooks/
│   ├── feature_engineering.ipynb  # Part 1 EDA · Part 2 feature engineering
│   └── modeling.ipynb             # §2 model selection · §3 explanations · §5 predictions · §6 operating point
└── predictions.csv                # deliverable 1
CLAUDE.md                          # problem brief + decisions captured during the work
README.md
```

---

## Data-quality note

Read every CSV with `keep_default_na=False, na_values=['']`. Otherwise pandas coerces region `NA` (North America, 77 members) to null — the data dictionary says only empty string is null. (Handled in `feature_engineering.ipynb` §1.2.)
