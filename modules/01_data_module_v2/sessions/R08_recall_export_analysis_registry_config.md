# R08 — Recall: Sessions 29–36 (Export Writers + Analysis + Registry + Config + Capstone)

**Date:** 2026-06-14
**Session number:** R08 of 08
**Purpose:** Final recall — consolidate Sessions 29–36, the full pipeline view
**Estimated study time:** 20 min
**Status:** 🟢 Complete

---

## §1 Modules Covered

| Session | Module | LOC |
|---|---|---|
| 29 | Export Writers | graph_writer 106, label_writer 150, metadata_writer 200, token_writer 95 |
| 30 | Analysis: Co-occurrence + Overlap | 187 + 267 |
| 31 | Analysis: Feature Dist + Balance + Drift | 436 + 134 + 298 |
| 32 | Registry: Dataset Diff + Lineage | 161 + 98 |
| 33 | Ingestion Freshness | 120 |
| 34 | Verification: Class Auditor + Report | 189 + 272 |
| 35 | Configuration | config.yaml 390 |
| 36 | Capstone | All modules |

**Total LOC covered:** ~3,393

---

## §2 Key Decisions Recap

### Export Writers (29)
- **Label writer** reads split JSONL only (Fix 1) — canonical input, not re-scattered label files
- **Metadata writer** recomputes `loc` and `n_functions` from .sol source (Fix 3) — split JSONL's `loc=0` is a placeholder
- **Graph/token writers** iterate in the same order (train → val → test from split JSONL) → aligned shard positions
- **Safe globals registration** needed for PyTorch `weights_only=True` with PyG types

### Analysis (30–31)
- **Two co-occurrence matrices:** directed (joint count) + conditional (P(B|A))
- **Two overlap matrices:** exact (same sha256) + near (same dedup_group)
- **6 features** for per-class analysis: node_count, edge_count, cyclomatic_complexity, call_depth, function_count, loc
- **Drift = KS test** (p < 0.01 → WARNING) on both features AND label rates
- **Complexity proxy risk** — the headline output that catches the "model learns contract size" failure

### Registry (32)
- **Dataset diff** compares two versions → per-class delta + heuristic F1 projection
- **Lineage** records every transformation step as a DAG → audit trail for bug reproduction
- **Lineage to DOT** — Graphviz visualization for debugging

### Freshness (33)
- **Two checks:** source pins vs upstream HEAD, installed Slither vs latest PyPI
- **Non-blocking:** informational report in `data/analysis/`
- **Run 9 motivation:** Slither API broke silently, corrupted graphs for 2 weeks

### Config (35)
- **390 lines** of YAML — the single source of truth for thresholds, sources, and pipeline behavior
- **6 critical-path sources** (3 enabled, 1 disabled, 2 have gaps)
- **Min viable corpus:** 4000 total, 300/100 per-class, 90% SmartBugs recall
- **Schema lock:** `pipeline.schema_version` must match `graph_schema.FEATURE_SCHEMA_VERSION` or pipeline errors

---

## §3 The 3 Hardest Problems — Final Answer Version

| Problem | Detection | Mitigation |
|---|---|---|
| **Label noise** (89.4% FP in BCCC) | 6-stage defense chain (tier → merger → semantic → FP → gate) | FAIL blocks export |
| **Dataset contamination** (same code in train+test) | 4-stage defense (dedup → enforcer → auditor → project splitter) | Reassignment + report |
| **Complexity proxy** (model learns "big=bad") | Feature dist analysis → `complexity_proxy_risk.md` | Control features, per-contract weighting |

---

## §4 All 7 Stages — Data Flow in One Diagram

```
sources_config.yaml
        │
        ▼
┌─────────────────────────────────────────────┐
│ Stage 1: INGEST + PREPROCESS                │
│ raw → flatten → compile → dedup → normalize │
│ → segment → manifest.json                   │
│ Output: preprocessed/<sha256>.sol + .meta   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Stage 2: GRAPH EXTRACTION                   │
│ .sol → Slither → call_graph + cfg + pdg     │
│ → graph_extractor → tokenizer               │
│ Output: representations/<sha256>.pt + tokens │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Stage 3: LABELING                           │
│ meta.json → parsers → merger                │
│ Output: labels/merged/<sha256>.labels.json   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ Stage 4: VERIFICATION                       │
│ class_auditor → semantic_checker →           │
│ fp_estimator → negative_checker → gate       │
│ Output: verification_report.md + verdicts    │
└──────────────────┬──────────────────────────┘
                   │ (gate passed?)
                   ▼
┌─────────────────────────────────────────────┐
│ Stage 5: SPLITTING                          │
│ stratified_splitter → dedup_enforcer →       │
│ leakage_auditor → nonvulnerable_cap          │
│ Output: splits/<run_id>/{train,val,test}.json│
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│ Stage 6: ANALYSIS│  │ Stage 7A: EXPORT │
│ balance → overlap│  │ chunk_export →   │
│ co-occurrence →  │  │ 4 writers →      │
│ feature_dist →   │  │ artifact_hash →  │
│ drift_monitor    │  │ manifest.json    │
│ freshness_report │  │ (Fix A)          │
│ (DVC, read-only) │  │                  │
└──────────────────┘  └──────────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │ Stage 7B: TRAINING  │
                    │ SentinelDataset     │
                    │ loads export shards │
                    │ by contract_id      │
                    └─────────────────────┘
```

---

## §5 Critical File Paths (Must Know)

```
data_module/config.yaml                           # source definitions + thresholds
data_module/sentinel_data/preprocessing/pipeline.py   # 5-step preprocessing
data_module/sentinel_data/representation/graph_schema.py  # feature definitions
data_module/sentinel_data/labeling/merger.py        # tier precedence
data_module/sentinel_data/verification/gate.py      # per-class verdicts
data_module/sentinel_data/splitting/splitters.py    # 4 split strategies
data_module/sentinel_data/export/chunker.py         # artifact hash Fix A
data_module/sentinel_data/analysis/feature_dist.py  # complexity proxy risk
```

---

## §6 Total Stats

| Metric | Value |
|---|---|
| Sessions written | 36 main + 8 recall = 44 files |
| Source modules covered | ~47 `.py` files |
| Total source LOC | ~22,000 |
| Pipeline stages | 7 (0–7B) |
| Critical-path sources | 6 (3 enabled) |
| Vulnerability classes | 10 (9 + NonVulnerable) |
| Verification tiers | 5 (T0–T4) |
| Split strategies | 4 (random, stratified, project, temporal) |
| Export artifact types | 4 (labels, metadata, graphs, tokens) |

---

## §7 Interview Challenge

**Question:** "Design a dataset versioning scheme for a 22K-contract, 10-class vulnerability corpus that ships a new version every 2 weeks."

**Answer:**

SENTINEL's approach:
1. **Naming**: `v2.0.0`, `v2.0.1`, `v2.1.0` (semver with no breaking = patch, new source = minor)
2. **Registry**: SQLite catalog with artifact_hash, schema_version, source digests, lineage DAG
3. **Export append-only**: each version is a new export directory; old versions aren't overwritten (DVC tracked)
4. **Split manifest**: per-version split metadata with contract_id lists → reproducible splits
5. **Drift monitor**: KS test between consecutive versions → `drift_report.md` for model team
6. **Dataset diff**: added/removed/changed contracts per class, with predicted F1 delta heuristic
7. **Changelog**: `changelog.md` records every registration — "added 1,200 SmartBugs Wild (T3), removed 12 FAIL contracts"
8. **Lineage**: every artifact's transformation DAG → "this graph was extracted by Slither 0.9.3 at preprocessing v1.2"
9. **Source pinning**: git commits pinned in config.yaml → freshness checker warns if upstream moved
10. **Schema lock**: `pipeline.schema_version` must match `graph_schema.FEATURE_SCHEMA_VERSION` → no silent feature drift between versions
