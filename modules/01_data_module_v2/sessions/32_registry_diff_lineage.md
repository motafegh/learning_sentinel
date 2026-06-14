# Session 32 — Registry: Dataset Diff + Lineage Tracker

**Date:** 2026-06-14
**Session number:** 32 of 36
**Mode:** Understand (dataset_diff 161L, lineage_tracker 98L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Registry / Catalog** (Session 04): SQLite + YAML mirror of preprocessing, labels, splits
- **Dataset versions** (Session 04): named, append-only, DVC-tracked
- **Split manifests** (Session 25): per-version metadata

---

## §1 Dataset Diff (161 LOC) — Version Comparison

Per AUDIT_PATCHES 5-P7: per-class metric projection.

### Structure (lines 28–60)

```python
@dataclass
class DatasetDiff:
    name_old, name_new: str
    added_contracts: list[str]
    removed_contracts: list[str]
    common_contracts: list[str]
    label_changes: list[dict]          # per-contract label diffs
    per_class: list[PerClassMetric]    # headline numbers

@dataclass
class PerClassMetric:
    class_name: str
    count_old, count_new: int
    delta_count, delta_pct: float
    tier_breakdown_old, tier_breakdown_new: dict
    predicted_f1_delta_pct: float     # coarse heuristic
```

### What It Reports

```
v1 → v2 dataset diff:

Added:    2,341 contracts
Removed:  12 contracts (dropped for verification FAIL)
Common:   19,000 contracts (label may have changed)

Per-class changes:
- Reentrancy:  200 → 450  (+125%)  predicted F1 Δ: +4.2%
- AccessControl: 50 → 120  (+140%)  predicted F1 Δ: +6.1%
- NonVulnerable: 18,000 → 22,000  (+22%)
```

### Predicted F1 Delta

A **coarse heuristic** (lines 35–37): if a class got more positives (especially from higher-tier sources), the model's F1 is expected to improve. Not a rigorous projection — just a directional signal for the model team: "Run 11 will likely be better for Reentrancy."

### Changelog

Each diff generates a `changelog.md` entry. Updated with every dataset version registration.

---

## §2 Lineage Tracker (98 LOC) — Audit Trail

Per D-5.5: every artifact has a DAG of transformations.

### Recording a Lineage Step (lines 24–40)

```python
def record_lineage_step(lineage: dict, step: str, **details) -> dict:
    """Append a step to a lineage dict.

    lineage = {
        "steps": [
            {"step": "ingest", "ts": "2026-06-14T...", "source": "solidifi"},
            {"step": "preprocess", "ts": "2026-06-14T...", "hash": "abc..."},
            {"step": "extract_graph", "ts": "2026-06-14T...", "n_nodes": 42},
        ],
        "parents": ["sha256_of_raw", "sha256_of_preprocessed"]
    }
    """
```

### Visualization (lines 43–52)

```python
def lineage_to_dot(lineage: dict) -> str:
    """Render lineage as Graphviz DOT for debugging."""
```

### Artifact Hashing (lines 55–60)

Thin wrapper around `catalog.compute_hash` for streaming SHA-256 of large artifacts.

### Who Consumes Lineage

- **Data team audit**: "Which preprocessing version produced this label's sidecar?"
- **Bug reproduction**: "Run 9's graph_extractor broke on contracts with lineage step `compiler: 0.8.0`. Did we re-run those?"
- **Reproducibility**: The full lineage DAG captures every transformation from raw source → export shard

---

## §0.6 Exercises (Q1–Q4)

**Q1: A model team member says "this contract's graph looks wrong." How does lineage help?**

The lineage trace shows every processing step: ingestion connector → compiler version → graph extractor hash → etc. The team can identify which step produced the corrupted output and whether the bug affects other contracts from the same step.

**Q2: Dataset diff shows `delta_pct = +2.5%` for Reentrancy but `predicted_f1_delta_pct = -1.0%`. How is this possible?**

More positives but lower-quality ones (e.g., new positives from T4 sources with high noise). The `predicted_f1_delta_pct` heuristic accounts for tier breakdown changes. The coarse heuristic might be wrong — the model team validates experimentally.

**Q3: How does the lineage tracker interact with the versioner (Session 21)?**

The versioner assigns version tags to artifacts in the registry catalog. The lineage tracker records processing steps WITHIN a version. Together they answer: "which steps produced artifact X at version Y."

**Q4: A contract's dedup_group changes between preprocessing runs. Does lineage capture this?**

Yes — each preprocessing run appends a lineage step with the dedup group assignment. Diffing lineage between runs shows the change. The `changelog.md` from dataset_diff would note any contracts whose dedup group changed.
