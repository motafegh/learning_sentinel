# Session 31 — Analysis: Feature Dist + Balance Viz + Drift Monitor

**Date:** 2026-06-14
**Session number:** 31 of 36
**Mode:** Understand (feature_dist 436L, balance_viz 134L, drift_monitor 298L)
**Estimated study time:** 25 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Graph sidecars** (Session 16): `.rep.json` has node_count, edge_count per contract
- **Merged labels** (Session 27): per-contract `.labels.json`
- **DVC tracked outputs**: analysis artifacts are read-only, version-immutable

---

## §1 Balance Viz (134 LOC) — Who Has What

Per D-6.1: "analysis is read-only, DVC-tracked outputs."

```python
@dataclass
class BalanceTable:
    per_class: dict            # Reentrancy → 450 positives
    per_source: dict           # solidifi → 600 positives
    per_tier: dict             # T0 → {Reentrancy: 200, ...}
    per_class_source: dict     # Reentrancy → {solidifi: 200, dive: 50, ...}
    multi_label_count: int     # contracts with ≥2 positives
```

**The 4 breakdowns answer these questions:**
1. **Per-class:** Which class has the fewest positives? (access control: ~50)
2. **Per-source:** Which source contributes the most? (SolidiFI: ~600)
3. **Per-tier T0→T4:** Are T4 sources dominating a class?
4. **Per-class-source:** Is a class overly dependent on one source?

### Outputs
```
data/analysis/<run_id>/
├── balance_table.csv     # flattened rows
└── balance_plot.png      # grouped bar chart
```

---

## §2 Feature Dist (436 LOC) — The Run-9-Failure Catcher

Per D-6.2: per-class feature distributions that caught the Run 9 failure.

### 6 Features (lines 33–36)

```python
DEFAULT_FEATURES = [
    "node_count", "edge_count",           # from .rep.json
    "cyclomatic_complexity", "call_depth", # from .sol source
    "function_count", "loc",              # from .sol source
]
```

### Per-class Stats (lines 40–56)

```python
@dataclass
class PerClassStats:
    feature_stats: dict   # {feature: {mean, std, min, max, median, n}}
```

**Per 6-P1:** Rank correlation between feature and precision. E.g., if Reentrancy contracts with few nodes have low gate precision, that's a signal.

**Per 6-P2:** Label-conditional feature distribution:
```python
label_conditional[feature][positive] = {mean, std, ...}
label_conditional[feature][negative] = {mean, std, ...}
```
If positive contracts are systematically larger than negatives, the model might learn "big contract = vulnerable" instead of the actual pattern.

### Complexity Proxy Risk (lines 59+)

```python
def _loc(sol_text):    # non-empty, non-comment lines
def _function_count(sol_text):  # count function definitions
```

The headline output: `complexity_proxy_risk.md`. If any class's positive contracts differ significantly from negatives in `node_count` or `loc`, the model risks learning complexity as a proxy for vulnerability → **complexity-proxy risk**.

### Outputs
```
data/analysis/<run_id>/
├── feature_dist_table.csv
├── feature_dist_plot.png
└── complexity_proxy_risk.md    # the headline
```

---

## §3 Drift Monitor (298 LOC) — Version-Update Gate

Per D-6.5: KS test for feature + label distribution drift between dataset versions.

### Two Drift Types (lines 5–11)

**1. Feature drift (FeatureKSResult):** KS test on numerical features (node_count, loc). Catches: "the contracts themselves got bigger/smaller."

**2. Label drift (LabelKSResult):** KS test on binary label rates. Catches: "the class balance shifted even if contracts look the same."

### Drift Report (lines 60+)

```python
@dataclass
class FeatureKSResult:
    feature: str
    n_baseline: int, n_new: int
    statistic: float, pvalue: float
    warning: bool
    insufficient_sample: bool

@dataclass
class LabelKSResult:
    class_name: str
    n_baseline_pos, n_new_pos, rate_baseline, rate_new
    statistic, pvalue, warning
```

If `pvalue < 0.05` AND sample is sufficient → **warning=True**. Warnings are recorded in the drift report.

### Use Case

```
v1 corpus → train model → deploy
v2 corpus (more sources, more contracts):
    drift_monitor(v1, v2):
    - Reentrancy positives went from 200→450 (label drift)
    - Average node_count went from 42→67 (feature drift)
    → Model team warned: retrain needed, v1 model won't generalize
```

### Output
```
data/analysis/<run_id>/drift_report.md
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: The drift monitor reports `feature=node_count, pvalue=0.001, warning=True`. What's the concern?**

The node_count distribution shifted significantly between versions. The old model trained on v1's smaller graph distribution may not generalize to v2's larger graphs. The retraining team should check if the feature drift is due to new sources with bigger contracts or an artifact of the extraction pipeline.

**Q2: Feature dist shows Reentrancy positives have `mean_loc=250, n=200` and negatives have `mean_loc=180, n=10000`. How does this affect the model?**

The 70-LoC gap means the model might learn "longer contracts = reentrancy" rather than actual reentrancy patterns. This is a **complexity-proxy risk** — flagged in `complexity_proxy_risk.md`. Mitigation: weighted loss by contract size, or add LoC as a control feature.

**Q3: Balance viz shows 90% of AccessControl positives come from SmartBugs Curated alone. Why is this risky?**

If SmartBugs Curated has systematic labeling errors for AccessControl, those errors dominate the class signal. The model learns SmartBugs' labeling bias rather than true AccessControl patterns. Fix: add more AccessControl sources or apply per-source loss weighting.

**Q4: The drift monitor says `insufficient_sample=True` for a class with 5 positives. Is the drift report actionable?**

No — KS test needs ~20+ samples per class for meaningful statistics. The `insufficient_sample` flag means the class is too rare to detect drift reliably. The report records it but the model team should treat it as a metadata note, not a decision signal.
