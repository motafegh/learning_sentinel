# Session 25 — Splitting (Stage 5)

**Date:** 2026-06-14
**Session number:** 25 of 36
**Mode:** Understand (splitters 441L, nonvulnerable_cap 163L, leakage_auditor 163L, dedup_enforcer 116L)
**Estimated study time:** 30 min teaching + 15 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Verified labels** (Session 23–24): merged `.labels.json` with per-class PASS/FAIL gate verdicts
- **SHA-256 identity** (Session 08): contracts are content-addressed; dedup groups from preprocessing
- **Graph representations** (Session 16): `.pt` files per contract

---

## §1 Architecture — Two-Pass Split (lines 23–27)

```
Pass 1: Splitter (splitters.py)
    Assigns contracts to train/val/test per strategy
    Output: "stratified splits"

Pass 2: Dedup Enforcer (dedup_enforcer.py)
    Reassigns near-dup groups that straddle splits
    Output: "leak-free splits"

Post-split: Leakage Auditor (leakage_auditor.py)
    Independent text-shingle similarity check
    Output: report (informational, non-blocking)

Post-split: NonVulnerable Cap (nonvulnerable_cap.py)
    Caps NonVulnerable class to N:1 ratio
    Default: 3:1 (configurable per class)
```

---

## §2 Four Splitter Strategies (lines 48–76, 441 total)

### `Contract` Dataclass (lines 56–75)

```python
@dataclass
class Contract:
    sha256: str
    source: str           # "solidifi" / "dive" / "smartbugs_curated" / "disl"
    tier: str             # "T0" / "T1" / "T2" / "T3" / "T4"
    classes: dict[str, int]  # {"Reentrancy": 1, "IntegerUO": 0, ...}
    primary_class: str = "NonVulnerable"
    n_pos: int = 0
    loc: int = 0                    # for temporal splitting
    year: int = 0                   # for temporal splitting
    dedup_group: Optional[str] = None  # for dedup_enforcer
    project_id: Optional[str] = None   # for project-level splitter
```

### Strategy 1 — Random (lines 48–49)

```python
random_splitter(contracts, ratios=(0.70, 0.15, 0.15), seed=42)
```

Simple random assignment. Deterministic via seed. Used as baseline or for sources where stratified isn't needed.

### Strategy 2 — Stratified (lines 50–51)

```python
stratified_splitter(contracts, ratios=(0.70, 0.15, 0.15), strata=["class", "tier"])
```

Per-class distribution preserved within ±2% of target. Also stratifies by confidence tier so T0/T1 sources aren't all in test. This is the **default strategy** per proposal §3.6.

### Strategy 3 — Project (lines 52–53)

```python
project_splitter(contracts, project_key="project_id")
```

Entire projects (e.g., all contracts from Bastet, ScaBench) kept in one split. Prevents project-level leakage where different functions from the same DApp appear in train and test. Used for audit datasets.

### Strategy 4 — Temporal (lines 54)

```python
temporal_splitter(contracts, cutoff_year=2023)
```

Pre-2023 contracts → train. Post-2023 → val/test. Used to simulate "future contract prediction" — train on older contracts, test on newer ones (the real-world deployment scenario).

### Per-Source Strategy (lines 13–14, config.yaml)

```yaml
splitting:
  strategies:
    defihacklabs: project   # project-level for audit datasets
    smartbugs_curated: project
    dive: stratified        # stratified with project-level fallback
    solidifi: stratified
```

---

## §3 Dedup Enforcer (116 LOC)

**Pass 2** of the two-pass split (lines 23–27). Reads the `dedup_group` field on each `Contract` and reassigns any group that straddles splits:

```
Input splits:  train: [A, B], val: [C]  ← A, B, C are in same dedup_group
                            ↓
Enforcer moves C to match the majority split:
Output splits: train: [A, B, C], val: []
```

**Security guarantee:** No SHA-256 identical or AST-similar (threshold 0.85) contract appears in >1 split. This prevents train→test leakage via copy-paste duplicates.

---

## §4 Leakage Auditor (163 LOC)

Independent post-split safety net using **text-shingle similarity** (D-5.3):

```python
SHINGLE_SIZE = 3  # character trigrams
THRESHOLD = 0.5   # Jaccard similarity threshold

def _shingles(s: str, n=3) -> set[str]:
    s = re.sub(r"\s+", " ", s.lower())
    return {s[i:i+n] for i in range(len(s)-n+1)}

def _text_similarity(a: str, b: str) -> float:
    sa, sb = _shingles(a), _shingles(b)
    return len(sa & sb) / len(sa | sb)  # Jaccard
```

**Difference from dedup_enforcer:**
- Dedup enforcer uses preprocessing's AST-based `dedup_group` to REASSIGN
- Leakage auditor uses TEXT SHINGLES (different method) to DETECT AND REPORT only
- If auditor finds leaks after enforcer → bug in enforcer

---

## §5 NonVulnerable Cap (163 LOC)

Addresses the 428:1 positive-to-negative ratio problem:

```
Total positives:  ~1,200
DISL negatives:  ~514,000
Ratio:           428:1  ← reproduces BCCC failure mode
```

**Solution:** Subsample Negatives to at most N:1 ratio:

```python
def apply_nonvulnerable_cap(splits, cap=3.0):
    """Cap NonVulnerable class to `cap` × total positives."""
    for split in [train, val, test]:
        nv = split.get_nonvulnerable()
        max_allowed = int(total_positives * cap)
        if len(nv) > max_allowed:
            # Subsample, stratified by source
            split.nonvulnerable = stratified_subsample(nv, max_allowed)
```

**Default cap: 3:1** (empirical sweet spot). Per-class override via config.

**Stratification by source:** Prevents the subsample from being 100% DISL (largest source). Per-source distribution is preserved.

**Audit log:** Every subsample recorded in `split_manifest.json` with original count, capped count, per-source breakdown.

---

## §6 Split Manifest Output

After both passes, a `split_manifest.json` is written:

```json
{
    "version": "v2.0.0",
    "created_at": "2026-06-14T12:00:00Z",
    "split_strategy": "stratified",
    "ratios": [0.70, 0.15, 0.15],
    "total_contracts": 10000,
    "splits": {
        "train": {"count": 7000, "nonvulnerable_capped": 600},
        "val": {"count": 1500, "nonvulnerable_capped": 300},
        "test": {"count": 1500, "nonvulnerable_capped": 300}
    },
    "dedup_enforcer": {"reassigned": 23},
    "leakage_auditor": {"leaks_found": 0, "text_threshold": 0.5},
    "nonvulnerable_cap": {"original_nv": 514000, "capped_nv": 1200}
}
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: A contract appears in both DIVE and SolidiFi with different sha256s but identical normalized content. Which splitter handles this?**

The dedup_enforcer (Pass 2). It uses the `dedup_group` field from preprocessing's deduplicator. If the two files have different sha256s but are in the same `dedup_group` (Level 3 normalized-text match), the enforcer reassigns them so they're in the same split. The leakage auditor catches any cases the enforcer misses.

**Q2: Why is the nonvulnerable cap applied AFTER splitting instead of BEFORE?**

The cap subsamples NonVulnerable contracts from each split independently (preserving per-split distribution). If applied before splitting, the stratified splitter would see a distorted class distribution. Applying after means the stratified splitter sees the real distribution, and the cap trims negatives without affecting positive allocation.

**Q3: The temporal splitter uses a cutoff_year=2023. How does it determine a contract's year?**

From the `year` field on the `Contract` dataclass. This is populated from the contract's `original_path` (which may encode a year, as in DeFiHackLabs' `2018-04/` directory) or from the `meta.json` `pragma_raw` (which implies a Solidity version year). If neither source is available, year=0 (treated as pre-2023).

**Q4: The leakage auditor finds 3 leaks after dedup_enforcer. What action is taken?**

The leaks are recorded in `split_manifest.json` for human review. The auditor does NOT block the split (that's the enforcer's job). The data team investigates whether the enforcer's AST-similarity threshold (0.85) missed a case that text-shingle similarity (0.5) caught. The fix is either: (a) lower the enforcer's threshold, or (b) add the missed pair to a manual blocklist.
