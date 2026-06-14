# ML09 — Calibration + Threshold Tuning

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML09 of 10
**Source:** `ml/scripts/calibrate_temperature.py` (258L), `ml/scripts/tune_threshold.py` (616L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Model output** (ML01): [B, 10] logits → sigmoid → [B, 10] probabilities
- **Validation set** (ML07): held-out split used for threshold tuning
- **Per-class thresholds** (ML08): predictor loads from `{checkpoint}_thresholds.json`

---

## §1 Calibrate Temperature (258 LOC) — Per-Class Temperature Scaling

### Problem

Run 4 ECE (Expected Calibration Error) = 0.205 to 0.310 across classes. A single global T cannot fix per-class miscalibration. Example: Reentrancy may be overconfident (predicts 0.8 when accuracy is 0.6) while IntegerUO is underconfident (predicts 0.3 when accuracy is 0.5).

### Solution

Fit one scalar T_c per class by minimising BCE NLL on the validation set:

```python
calibrated_logit_c = logit_c / T_c
```

### ECE Computation (lines 44–62)

```python
def compute_ece(probs, labels, n_bins=15):
    """Equal-width ECE for a single class."""
    bins = np.linspace(0.0, 1.0, n_bins + 1)
    ece = 0.0
    for lo, hi in zip(bins[:-1], bins[1:]):
        mask = (probs >= lo) & (probs < hi)
        if not mask.any(): continue
        conf = probs[mask].mean()
        acc  = labels[mask].mean()
        ece += mask.mean() * abs(conf - acc)
    return ece
```

### Output

```json
{
    "Reentrancy": 1.24,
    "IntegerUO": 0.87,
    "Timestamp": 1.52,
    ...
}
```

T > 1.0 = model is overconfident (scale logits DOWN). T < 1.0 = model is underconfident (scale logits UP).

Also produces: ECE before/after per-class bar chart + JSON stats.

---

## §2 Tune Threshold (616 LOC) — Per-Class F1 Optimizer

### What It Does

1. Loads the validation split
2. Loads the trained checkpoint
3. Runs one forward pass to collect sigmoid probabilities for all 10 classes
4. Sweeps thresholds per class on cached probabilities (0.01 to 0.99, step 0.01)
5. Picks the threshold that maximizes per-class F1
6. Saves `{checkpoint.stem}_thresholds.json`

### Sweep Logic

```python
@dataclass(frozen=True)
class SweepRow:
    threshold: float
    f1: float
    precision: float
    recall: float
    predicted_positives: int

for cls in range(10):
    for thresh in np.arange(0.01, 1.0, 0.01):
        preds = (probs[:, cls] >= thresh).astype(int)
        f1 = f1_score(labels[:, cls], preds)
        # store SweepRow
    best = max(rows, key=lambda r: r.f1)
    thresholds[CLASS_NAMES[cls]] = best.threshold
```

### Why Per-Class (not global)

Old version: one global threshold (e.g., 0.5). Problem: minority classes (UnusedReturn, ExternalBug) have probability mass centered at 0.35-0.50. A 0.5 threshold gives them F1 ≈ 0. A 0.35 threshold gives F1 ≈ 0.30. A single threshold can't serve both majority and minority classes.

### Fixes Applied

- **Fix #3**: `load_model_from_checkpoint()` passes dropout, lora_target_modules from saved config
- **Fix #5**: DataLoader kwargs conditional on `num_workers > 0` — eliminates PyTorch 2.x UserWarning about `prefetch_factor=None`

---

## §3 How They Work Together

```
train.py → checkpoint.pt
    │
    ├──→ calibrate_temperature.py → temperatures.json
    │     (applied at inference: logit / T before sigmoid)
    │
    └──→ tune_threshold.py → thresholds.json
          (applied at inference: prob >= threshold → positive)
```

During inference (predictor.py):

```python
logits = model(...)
for cls in range(10):
    logits[cls] /= temperatures[cls]       # calibrate
probs = torch.sigmoid(logits)
preds = {cls: probs[i] for i, cls in enumerate(CLASS_NAMES)
         if probs[i] >= thresholds[cls]}   # threshold
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: After temperature scaling, Reentrancy's T=1.52. What does this mean?**

The model is overconfident for Reentrancy — it outputs probabilities that are too high relative to actual accuracy. The logits are divided by 1.52 before sigmoid, pulling probabilities toward 0.5. The expected ECE improvement: from ~0.31 to ~0.10.

**Q2: tune_threshold.py finds UnusedReturn has optimal threshold at 0.12. Is this correct?**

Yes — UnusedReturn is the rarest class (~50 positives). Its probability mass clusters at 0.1-0.2. A threshold of 0.5 would find zero positives (F1=0). 0.12 is the F1-optimal tradeoff for a rare class. The classifier head learned to assign low probabilities to rare classes — the threshold compensates.

**Q3: The threshold JSON file is missing for a new checkpoint. What does the predictor do?**

It falls back to `self.threshold` (default 0.5) and logs a warning. All classes use the same 0.5 threshold. Expected: minority classes get F1 ≈ 0. The operator should run `tune_threshold.py` before promoting.

**Q4: Calibration improves ECE but does it change accuracy?**

Temperature scaling does NOT change accuracy — it preserves the ranking of logits (dividing by a positive scalar preserves the argmax). It only changes the CONFIDENCE associated with each prediction. ECE drops while accuracy stays the same.
