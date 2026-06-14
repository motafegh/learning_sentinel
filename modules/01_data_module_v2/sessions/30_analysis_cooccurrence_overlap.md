# Session 30 — Analysis: Co-occurrence + Overlap Detector

**Date:** 2026-06-14
**Session number:** 30 of 36
**Mode:** Understand (cooccurrence 187L, overlap_detector 267L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Merged labels** (Session 27): per-contract `.labels.json` with multi-class support
- **Dedup groups** (Session 09): AST-based near-dup clustering at 0.85 threshold
- **BCCC 99% co-occurrence** (Session 22): DoS + Reentrancy co-label noise

---

## §1 Co-occurrence (187 LOC) — Directed + Conditional Matrices

Per AUDIT_PATCHES 6-P4 (C-10): two matrices.

### Directed matrix (lines 33–37)

```python
directed[a][b] = count of contracts where both a and b are positive
```

This is the joint count P(a=1 ∧ b=1) × N. Symmetric in practice but stored in directed form for clarity.

### Conditional matrix (lines 37–41)

```python
conditional[a][b] = directed[a][b] / count_positive(a)
```

This is P(b=1 | a=1) — the conditional probability that if class A is present, class B is also present.

### Flagged pairs (flag_threshold=0.50)

If any conditional[a][b] > 0.50, the pair is flagged. This catches the BCCC 99% DoS↔Reentrancy pattern. The flag threshold is conservative — 50% is far from BCCC's 99% but catches egregious co-label noise.

### Outputs

```
data/analysis/<run_id>/
├── cooccurrence_matrix.csv      # directed + conditional
└── cooccurrence_heatmap.png     # side-by-side heatmaps
```

---

## §2 Overlap Detector (267 LOC) — Exact + Near Jaccard

Per AUDIT_PATCHES 6-P5: distinguish EXACT overlap (same sha256) from NEAR overlap (AST-similar but different sha256).

### Exact Overlap (lines 40–73)

```python
def _index_labels_by_source(labels_root):
    """Build a map: source -> set of sha256s"""
    by_source: dict[str, set[str]] = defaultdict(set)
    # From merged labels and per-source label files
```

Jaccard similarity between sources:
```
J(A, B) = |A ∩ B| / |A ∪ B|
```

A high exact Jaccard means two sources share many of the same contracts.

### Near Overlap (lines 76+)

```python
def _index_dedup_groups(preproc_root):
    """Build a map: dedup_group_id -> set of sha256s"""
```

Near overlap uses AST-based dedup groups (threshold 0.85). If source A has sha256_x and source B has sha256_y in the same dedup group, they're near-overlaps.

### Why distinguish exact vs. near?

| Overlap type | Impact | Mitigation |
|---|---|---|
| Exact (same sha256) | Double-counting in loss — the same contract appears twice | Dedup enforcer moves one to match |
| Near (AST-similar, different sha256) | Training+test similarity — model sees near-identical code in both splits | Leakage auditor reports; threshold tuning |

### Outputs

```
data/analysis/<run_id>/
├── overlap_matrix.csv           # exact + near Jaccard
└── overlap_heatmap.png          # side-by-side heatmaps
```

---

## §3 How the Two Modules Interact

```
Merged Labels
     │
     ├──→ Co-occurrence: P(b=1|a=1) per pair
     │    Flags co-label noise (BCCC pattern)
     │
     └──→ Overlap Detector: J(A,B) per source pair
          Flags dataset contamination (exact + near)
```

Both are inputs to per-source loss weighting in the training pipeline (D-6.4).

---

## §0.6 Exercises (Q1–Q4)

**Q1: The co-occurrence matrix shows `conditional[DoS][Reentrancy] = 0.85`. What does this mean?**

85% of contracts labeled DoS-positive are ALSO labeled Reentrancy-positive. This is far above the 50% flag threshold, so the pair is flagged. The flag feeds into the verification gate and the per-source loss weighting.

**Q2: The overlap detector reports `exact_jaccard[solidifi][dive] = 0.35`. Is this high or low?**

0.35 Jaccard means 35% overlap. Given SolidiFI (~600 contracts) and DIVE (~200 contracts), this translates to roughly 70 shared contracts. This is significant — it means the model would see the same contracts from both sources, which the merger handles via tier precedence.

**Q3: Two sources have near_jaccard=0.60 but exact_jaccard=0.05. What's happening?**

They share almost no identical sha256s (low exact) but many AST-similar contracts (high near). This means the two sources have the same contracts compiled with different compilers or with minor modifications. The dedup enforcer's 0.85 threshold is catching many pairs, but some may leak across splits.

**Q4: Can the co-occurrence module be run on a subset of sources only?**

Yes — the `build_cooccurrence_matrices()` function takes any `labels_dir` that has `.labels.json` files. It's source-agnostic. You can pass `data/labels/merged/` for the full corpus or `data/labels/dive/` for DIVE-only analysis.
