# Session 24 — Verification: Semantic Checker + FP Estimator + Probe

**Date:** 2026-06-14
**Session number:** 24 of 36
**Mode:** Understand (semantic checker 283L, fp_estimator 336L, negative_checker 235L, probe 587L)
**Estimated study time:** 30 min teaching + 15 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Verification gate** (Session 23): consumes semantic_checker, fp_estimator, negative_checker outputs
- **Graph features** (Session 17): CEI path, block globals, unchecked detection — used by semantic checker
- **Label tiers** (Session 22): T0–T4 confidence hierarchy

---

## §1 Semantic Checker (283 LOC)

Validates labels against graph-extracted structural features. For example: a Reentrancy label is semantically consistent if the graph has a CEI path (external call before state write).

### Check Types

| Check | What it verifies | Example inconsistency |
|---|---|---|
| CEI path | Reentrancy label → graph.has_cei_path == 1 | Labeled Reentrancy but no CEI path |
| Block globals | Timestamp/TOD label → uses_block_globals > 0 | Labeled Timestamp but no block.* reads |
| Unchecked block | IntegerUO label → in_unchecked_block > 0 | Labeled IntegerUO, all nodes checked |
| External call | All "call"-based labels → external_call_count > 0 | Labeled ExternalBug, zero external calls |

### Per-Class Pass Rate

```python
ClassCheckSummary:
    class_name
    checkable: int        # contracts with available graph reps
    passed: int           # semantically consistent
    failed: int           # semantically inconsistent
    pass_rate: float      # passed / checkable
```

**What happens to failed contracts?** They're NOT removed from the dataset. The pass_rate is a CORPUS-LEVEL quality metric. If pass_rate is low, the class label quality is suspect. Individual failures are recorded for Stage 6 human review.

---

## §2 FP Estimator (336 LOC)

Estimates false positive rate by running detectors on a **negative set** (contracts NOT labeled for a given class).

### Logic

```
For each class C:
    1. Collect contracts NOT labeled with C (negatives)
    2. Run Slither detectors for C on negatives
    3. If Slither fires on a negative → potential FP signal
    4. FP rate = contracts_with_detector_fired / total_negatives_checked
```

**FP_RATE_FAIL_THRESHOLD = 30%** — if fp_rate > 30% for a class, the gate downgrades it.

**Limitation:** Only as good as the negative set. If the negative set has hidden positives (noise from other sources), the FP estimate is inflated.

---

## §3 Negative Checker (235 LOC)

The opposite of FP Estimator: checks if Slither fires on NON-labeled contracts for detectors that SHOULD fire.

### Logic

```
For contracts NOT labeled for any class:
    Run all Slither detectors
    If a detector fires → potential missed vulnerability
    Corpus-level status: OK / WARN / FAIL (based on fraction of contracts with missed signals)
```

**Purpose:** Detects systematic under-labeling. If 20% of "non-vulnerable" contracts have Slither's `reentrancy-eth` firing, the dataset is missing reentrancy labels.

**Integration:** The negative_checker status is a CORPUS-LEVEL signal (not per-class). A FAIL status adds a hard gate entry keyed as `__neg_check__` which blocks export.

---

## §4 Probe Dataset + Probe Trivials (587 LOC total)

Two modules for dataset quality probing:

### Probe Dataset (361 LOC)

Reads positive contracts for a class and reports structural properties:
- Contract size distribution
- Feature vector statistics (mean, std per dim)
- CEI path rate per class
- Graph size (n_nodes, n_edges) per source

### Probe Trivials (226 LOC)

Detects "trivial" positives — contracts that are trivially detectable by a simple pattern (not a learned model). A high rate of trivial positives suggests the label noise is low (easy patterns are well-covered). Used for:
- Sanity-checking label quality
- Measuring label noise at the "easy" end of the difficulty spectrum
- Identifying classes where detectors are redundant with simple heuristics

---

## §0.6 Exercises (Q1–Q4)

**Q1: The semantic checker finds a Reentrancy label with `has_cei_path=0`. Is this label wrong?**

Not necessarily. The contract might use a reentrancy pattern that doesn't fit the "CALL → WRITE" CEI definition (e.g., cross-function reentrancy, read-only reentrancy). The semantic check is a CONSERVATIVE filter: a missing CEI path is suspicious but not conclusive. The gate uses aggregate pass_rate, not per-contract decisions.

**Q2: The FP estimator computes fp_rate = 40% for Reentrancy from a T3 source. What happens?**

The gate sees fp_rate > 30% (FP_RATE_FAIL_THRESHOLD). Reentrancy from the T3 source gets a FAIL verdict, blocking export. The solution: review the T3 source's Reentrancy labels, remove false positives, re-run labeling and verification.

**Q3: The negative checker fires on 5% of non-labeled contracts. Is this a FAIL?**

5% is below the FAIL threshold (exact threshold is config-dependent, typically 10-15%). The status would be WARN — flagged for human review but not blocking export. A 5% missed-detector rate suggests minor under-labeling, not systematic noise.

**Q4: Probe trivials finds that 80% of Reentrancy positives are trivially detectable (single external call followed by state write). Is this good or bad?**

Both. Good: label noise is low (easy patterns are well-covered). Bad: the model may overfit to trivial patterns and miss real-world reentrancy that uses cross-function or read-only patterns. The probe result suggests augmenting training with non-trivial reentrancy examples.
