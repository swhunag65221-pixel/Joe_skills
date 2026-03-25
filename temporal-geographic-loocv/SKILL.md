---
name: temporal-geographic-loocv
description: Use when designing cross-validation for spatio-temporal data (e.g., epidemiology, climate, IoT sensors) where both time and geographic units matter. Use when building LOOCV pipelines that must avoid data leakage across time periods or spatial units, when splitting train/validation/test by date boundaries instead of row indices, or when combining Optuna hyperparameter search with leave-one-location-out validation.
---

# Temporal-Geographic LOOCV

## Overview

Leave-One-Location-Out Cross-Validation with strict temporal boundaries. Designed for datasets where observations are indexed by **(date, location)** and future data must never leak into training.

**Core principle:** Every data split — scaler fitting, unsupervised pretraining, supervised training, early-stopping, validation — must use **date-based boundaries**, never row-index slicing.

## When to Use

- Spatio-temporal prediction (dengue, air quality, crop yield, crime, sensor networks)
- Panel data with geographic units observed over time
- Any CV where train/test split must respect both temporal order and spatial independence
- Hyperparameter search (Optuna) over geographic units with temporal holdout

**When NOT to use:**
- Pure time-series with a single entity (use standard temporal CV)
- Cross-sectional data without time dimension (use standard k-fold)
- Data where spatial autocorrelation is negligible

## Semi-Annual Boundary Convention

All temporal splits use fixed **January 31** and **July 31** boundaries, creating 6-month windows. This is not arbitrary — it aligns with seasonal cycles (e.g., dengue wet/dry seasons, agricultural planting cycles).

### Fold Structure

```
Fold 1: train ≤ 2022-01-31 │ val 2022-02-01 ~ 2022-07-31 │ test 2022-08-01 ~ 2023-01-31
Fold 2: train ≤ 2022-07-31 │ val 2022-08-01 ~ 2023-01-31 │ test 2023-02-01 ~ 2023-07-31
Fold 3: train ≤ 2023-01-31 │ val 2023-02-01 ~ 2023-07-31 │ test 2023-08-01 ~ 2024-01-31
  ...each fold slides forward 6 months...
Fold N: train ≤ YYYY-07-31 │ val YYYY-08-01 ~ YYYY+1-01-31
```

**Key rules:**
- Every window spans exactly **6 months** (Feb-Jul or Aug-Jan)
- Boundaries are always the **last day** of January or July
- Train accumulates all history up to the boundary (expanding window)
- Validation/test are the next 6-month window after the boundary

### LOOCV Split Procedure (Step by Step)

For each target geographic unit V:

```
Step 1: Define boundaries (semi-annual)
        TRAIN_END  = 2025-07-31
        VAL_START  = 2025-08-01
        VAL_END    = 2026-01-31

Step 2: Scaler fit
        All rows ≤ TRAIN_END (all units, including V)

Step 3: Unsupervised pretrain data
        All rows ≤ TRAIN_END (all units, including V — no labels used)

Step 4: Supervised labeled data
        Labeled rows ≤ TRAIN_END, EXCLUDING unit V
        → date-based 90/10 split into Train / Early-Stop

Step 5: Validation data
        Labeled rows VAL_START ~ VAL_END, ONLY unit V

Step 6: Repeat steps 2-5 for each of the N target units
```

### The Date-Boundary Rule (Train/ES Sub-Split)

Within step 4, the 90/10 train/ES split must also use **date boundaries**, not row indices.

**Problem: Row-Index Splitting Leaks Dates**

```python
# ❌ WRONG: Row-index split can put same date in both train and ES
df_sorted = df.sort_values('date').reset_index(drop=True)
n_train = int(len(df_sorted) * 0.9)
train = df_sorted.iloc[:n_train]      # May contain 2025-01-22
es    = df_sorted.iloc[n_train:]      # May also contain 2025-01-22
```

Multiple locations share the same date. Row-index slicing puts some rows of date X in train and others in ES — **temporal leakage**.

**Solution: Date-Based Boundary**

```python
# ✅ CORRECT: Find the date at the 90th percentile, split on it
df_sorted = df.sort_values('date').reset_index(drop=True)
split_idx = int(len(df_sorted) * 0.9)
split_date = df_sorted.iloc[split_idx]['date']

train = df_sorted[df_sorted['date'] < split_date]    # All of date X in one set
es    = df_sorted[df_sorted['date'] >= split_date]    # No date overlap
```

**Verify zero overlap:**
```python
train_dates = set(train['date'].unique())
es_dates = set(es['date'].unique())
assert len(train_dates & es_dates) == 0, "Date leak detected!"
```

## Architecture: Multi-Process LOOCV

### Why Separate Processes

Each geographic unit runs as an independent process (not threads):
- **GPU memory isolation** — each process gets its own CUDA context
- **Fault tolerance** — one failed unit doesn't kill others
- **Resumability** — re-run only failed units
- **Parallelism** — 3 units per batch on a single GPU

### File Structure

```
train_loocv_village.py      # Single-unit Optuna search (called via nohup)
launch_loocv.sh             # Batch launcher (N units parallel)
collect_loocv_results.py    # Pool results + calibration + threshold
train_loocv_production.py   # Final model with best params
```

### Launcher Pattern

```bash
#!/bin/bash
UNITS=("unit_A" "unit_B" "unit_C" "unit_D" "unit_E")
MAX_PARALLEL=3

for ((i=0; i<${#UNITS[@]}; i+=MAX_PARALLEL)); do
    batch=("${UNITS[@]:i:MAX_PARALLEL}")
    pids=()
    for u in "${batch[@]}"; do
        nohup python train_unit.py --unit "$u" \
            > "logs/${u}_nohup.log" 2>&1 &
        pids+=($!)
    done
    for pid in "${pids[@]}"; do wait $pid; done
done
```

**Log separation:** Python logger writes to `{unit}.log`, nohup redirects stderr to `{unit}_nohup.log`. Never let both write to the same file.

## Data Split Diagram

```
Timeline: ──────────────────────────────────────────────────────────►

         │◄──── Expanding train window ────►│◄── 6-month val ──►│
         │                                  │                    │
         │  Scaler fit (all units)          │                    │
         │  Pretrain (all units)            │                    │
         │  Supervised (excl. unit V)       │  Validation        │
         │                                  │  (only unit V)     │
         ├──────────────────────────────────┼────────────────────┤
         │          ≤ 2025-07-31            │ 2025-08-01         │
         │                                  │    ~ 2026-01-31    │
         └──────────────────────────────────┴────────────────────┘
                        ▲                              ▲
                   Semi-annual                    Semi-annual
                   boundary (7/31)                boundary (1/31)

Within training period (supervised labeled data, excl. V):
         ┌────── Train (~90%) ──────┬──── ES (~10%) ────┐
         │     < split_date         │  >= split_date     │
         └──────────────────────────┴────────────────────┘
         (split_date = date at 90th percentile position)
```

**V = target geographic unit (left out)**

## Data Leakage Checklist

| Component | Correct Scope | Common Mistake |
|-----------|--------------|----------------|
| Scaler fit | All rows ≤ TRAIN_END | Including validation period |
| Unsupervised pretrain | All rows ≤ TRAIN_END | Including future rows |
| Supervised train | Labeled ≤ TRAIN_END, **excluding target unit** | Including target unit |
| Early-stop set | Labeled ≤ TRAIN_END, late temporal slice | Random split |
| Validation | Target unit only, VAL_START ~ VAL_END | Including other units |
| Train/ES boundary | Date-based (`< split_date` / `>= split_date`) | Row-index (`iloc`) |

### Scaler Fit Scope

```python
# ✅ Scaler sees only training period
scaler = MinMaxScaler()
scaler.fit(df[df['date'] <= TRAIN_END][feature_cols].values)

# ❌ Scaler sees validation period — leaks distributional info
scaler.fit(df[feature_cols].values)
```

### Target Unit Exclusion

```python
# ✅ Supervised training excludes the unit being validated
df_train = df_labeled[
    (df_labeled['date'] <= TRAIN_END) &
    (df_labeled['location'] != target_unit)
]

# ✅ Validation is ONLY the target unit in the holdout period
df_val = df_labeled[
    (df_labeled['date'] >= VAL_START) &
    (df_labeled['date'] <= VAL_END) &
    (df_labeled['location'] == target_unit)
]
```

## Optuna Integration

### Per-Unit Search → Unified Parameters

Each unit runs independent Optuna trials. After all units complete:

```python
# Strategy: pick params from the unit with highest AUC
# (with continuous search spaces, duplicate param sets are unlikely)
best = max(results, key=lambda r: r['best_auc'])
unified_params = best['best_params']
```

### Retrain with Best Params

After Optuna selects best params, **retrain once** with those params and collect validation predictions for calibration:

```python
val_probs, val_auc = retrain_best_and_predict(data, best_params)
result = {
    'unit': unit_name,
    'best_params': best_params,
    'best_auc': val_auc,
    'val_y_true': y_val.tolist(),    # For pooled calibration
    'val_y_prob': val_probs.tolist(),
    'status': 'done',
}
```

## Calibration: Pooling LOOCV Predictions

After all units complete, pool their out-of-sample predictions:

```python
# Pool all units' validation predictions
all_y_true, all_y_prob = [], []
for r in results:
    all_y_true.extend(r['val_y_true'])
    all_y_prob.extend(r['val_y_prob'])

y_pool = np.array(all_y_true)
p_pool = np.array(all_y_prob)

# 1. Pooled AUC (honest — each prediction is out-of-sample)
pooled_auc = roc_auc_score(y_pool, p_pool)

# 2. Platt scaling
platt = LogisticRegression(C=1e10, solver='lbfgs')
platt.fit(p_pool.reshape(-1, 1), y_pool)

# 3. Threshold optimization (Youden Index)
calibrated = platt.predict_proba(p_pool.reshape(-1, 1))[:, 1]
# J = sensitivity + specificity - 1, maximized
```

**Save the fitted Platt calibrator via joblib**, not by reconstructing from coefficients:

```python
# ✅ Save fitted object — preserves all internal state
joblib.dump(platt, 'platt_calibrator.joblib')

# ❌ Fragile: manually setting coef_/intercept_ on unfitted LR
lr = LogisticRegression()
lr.classes_ = np.array([0, 1])
lr.coef_ = np.array([[saved_coef]])     # May break across sklearn versions
lr.intercept_ = np.array([saved_intercept])
```

## Production Model Training

After calibration, train the final model on **all available data** up to the next semi-annual boundary:

```python
# Production boundary = next semi-annual date after validation window
# LOOCV val ended at 2026-01-31 → production uses everything ≤ 2026-01-31
PRODUCTION_TRAIN_END = '2026-01-31'

# Scaler: fit on ALL rows ≤ PRODUCTION_TRAIN_END (all units)
# Pretrain: ALL rows ≤ PRODUCTION_TRAIN_END (unsupervised, all units)
# Supervised: ALL labeled ≤ PRODUCTION_TRAIN_END (90/10 date-based split, all units)
# Calibrator: use the pooled LOOCV Platt calibrator (already fitted)
```

The production model uses **all geographic units** (no exclusion) and **all data up to the boundary** because it's the final deployment model. The train/ES sub-split still uses date boundaries (not row indices).

## Common Mistakes

### 1. Same log file for nohup and Python logger
```bash
# ❌ Both write to same file — garbled output
nohup python train.py --unit "$u" > "logs/${u}.log" 2>&1 &
# Python's setup_logger() also writes to logs/${u}.log

# ✅ Separate files
nohup python train.py --unit "$u" > "logs/${u}_nohup.log" 2>&1 &
```

### 2. Forgetting to save val_y_true/val_y_prob
Without per-unit predictions, you can't do pooled calibration. Each unit must save ground truth and predicted probabilities.

### 3. Reconstructing calibrator from coefficients
Sklearn's `LogisticRegression` has internal state beyond `coef_` and `intercept_`. Always `joblib.dump/load` the fitted object.

### 4. Using F2 threshold when Youden is intended
F2 favors recall (penalizes false negatives). Youden (J = sens + spec - 1) balances both. Choose based on domain cost:
- **Youden**: balanced sensitivity/specificity
- **F2**: when missing positives is costly (e.g., disease surveillance)

### 5. Missing `cal_samples` in meta
If downstream inference expects a `cal_samples` field in model metadata, include it even if calibration is done externally (set to 0).

## Quick Reference

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| TRAIN_END | Domain-specific | Last date for training data |
| VAL_START | TRAIN_END + 1 day | Start of validation window |
| VAL_END | Domain-specific | End of validation window |
| Train/ES ratio | 90/10 | Within training period, date-based |
| MAX_PARALLEL | 3 | Units per GPU batch |
| N_TRIALS | 20-50 | Optuna trials per unit |
| Platt C | 1e10 | Near-unconstrained logistic regression |
