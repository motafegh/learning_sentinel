# Session 36 — Pipeline End-to-End Summary + Capstone

**Date:** 2026-06-14
**Session number:** 36 of 36
**Mode:** Synthesize (all modules)
**Estimated study time:** 30 min teaching + 30 min Q&A
**Status:** 🟢 Complete

---

## §1 The Big Picture — 7 Stages

```
Stage 0: SOURCES (config.yaml)
   6 critical-path sources → 3 connectors (git, manual, etherscan)

Stage 1: INGESTION + PREPROCESSING
   fetch → flat → compile → dedup → normalize → segment
   Output: data/preprocessed/<source>/<sha256>.sol + .meta.json
   Key: SHA-256 identity, dedup groups, ingesting_manifest.json gate

Stage 2: GRAPH EXTRACTION
   call_graph → cfg_builder → pdg_builder → graph_extractor → tokenizer
   Output: data/representations/<source>/<sha256>.pt + .tokens.pt + .rep.json
   Key: CEI path, block globals, 6 graph features, cache manager

Stage 3: LABELING + VERIFICATION
   label parsers → merger → class_auditor → semantic_checker
   → tool_validator → fp_estimator → negative_checker → gate
   Output: data/labels/merged/<sha256>.labels.json
   Key: Tier precedence (T0>T1>T2>T3>T4), co-occurrence flagging

Stage 4: VERIFICATION GATE
   VERIFIED / PROVISIONAL / BEST-EFFORT / FAIL per class
   Gate must PASS to continue. FAIL classes blocked from export.

Stage 5: SPLITTING
   stratified_splitter → dedup_enforcer → leakage_auditor → nonvulnerable_cap
   Output: data/splits/<run_id>/{train,val,test}.jsonl
   Key: Two-pass split, 3:1 NonVulnerable cap, project-level for audits

Stage 6: ANALYSIS
   balance_viz → cooccurrence → feature_dist → drift_monitor
   → overlap_detector → freshness_checker
   Output: data/analysis/<run_id>/*.csv, *.png, *.md
   Key: Complexity-proxy risk, drift detection, BCCC co-occurrence

Stage 7A: EXPORT
   chunk_export → 4 writers → artifact_hash → manifest.json
   Output: data/exports/<version>/{labels,metadata}.parquet + shards

Stage 7B: TRAINING (ML side)
   SentinelDataset wraps SentinelDatasetExport
   __getitem__ → load shard → slice graph/tokens → return (graph, tokens, label)
```

---

## §2 The 3 Hardest Problems (and How SENTINEL Solves Them)

### Problem 1: Label Noise (BCCC had 89.4% FP)

**SENTINEL's defense chain:**
1. **Source tiering** — T0 (mathematical ground truth) → T4 (noisy labels)
2. **Tier conflict resolution** — highest tier wins
3. **Co-occurrence flagging** — T3/T4 sources with high DoS+Reentrancy flagged
4. **Semantic checker** — graph-structural consistency per class
5. **FP estimator** — Slither detector firing on negative set
6. **Gate** — FAIL = blocked from export

### Problem 2: Dataset Contamination (Same code in train and test)

1. **Dedup 3-level** — SHA-256 → AST → normalized text at preprocessing
2. **Dedup enforcer** — reassigns near-dup groups across splits
3. **Leakage auditor** — independent text-shingle check
4. **Project-level splitter** — entire DApps stay in one split

### Problem 3: Complexity Proxy (Model learns "long contract = vulnerable")

1. **Feature dist** — per-class feature distributions computed
2. **Complexity proxy risk** — flagged if positives differ from negatives
3. **Label-conditional features** — enables analysis before training

---

## §3 All 36 Sessions at a Glance

| Session | Module | Key Insight |
|---|---|---|
| 01 | Orientation | The 10-class taxonomy |
| 02 | CLI | Click dispatch tree |
| 03 | DB init | SQLite + YAML mirror |
| 04 | Registry / Catalog | Dataset versioning |
| 05 | Sync / Ingestion | Manifest as hard gate |
| 06 | Source Fetchers | 5 connectors abstracted |
| 07 | Label Folderize | DIVE's symlink trick |
| 08 | Preprocessing Pipeline | 5 steps in order |
| 09 | Deduplicator | 3-level cascade |
| 10 | Compiler | Two-pass version resolution |
| 11 | Flattener | Cascading fallback |
| 12 | Normalizer | Comment stripping |
| 13 | Segmenter | Bucket distribution |
| 14 | Parallel Pipeline | ThreadPoolExecutor |
| 15 | Graph Schema | v9 → v10 seam swap |
| 16 | Graph Extractor | Slither intermediary |
| 17 | Feature Engineering | 6 features |
| 18 | CFG + Edges | Node-to-edge |
| 19 | CFG Builder | Standalone module |
| 20 | Tokenizer + Orchestrator | 4-window sliding |
| 21 | Cache + Versioner | SHA-keyed persistence |
| 22 | Labeling Schema | 9+1 classes, tiers |
| 23 | Verification | Slither runner, validator, gate |
| 24 | Semantic Checker + FP + Probe | Graph-label consistency |
| 25 | Splitting | Two-pass + 3:1 cap |
| 26 | Label Parsers | Crosswalk per source |
| 27 | Label Merger | Tier precedence |
| 28 | Export + Chunker | Fix A (manifest last) |
| 29 | Export Writers | Parquet + sharded .pt |
| 30 | Co-occurrence + Overlap | Matrices for loss weighting |
| 31 | Feature Dist + Drift | Complexity proxy, KS tests |
| 32 | Dataset Diff + Lineage | Version comparison |
| 33 | Freshness Checker | Slither version guard |
| 34 | Class Auditor + Report | Unified verification |
| 35 | Config.yaml | The threshold hub |
| 36 | End-to-End | The big picture |

---

## §4 Q&A — The 10 Interview Questions

**Q1: What makes SENTINEL different from SmartBugs Wild or Slither-audited datasets?**

Tiered provenance with guardrails: T0 sources (SolidiFI's injection, mathematically guaranteed) anchor the label space. T3/T4 sources contribute scale but their labels are VERIFIED by gate before reaching training. No other dataset has multi-source verification with a per-class gate.

**Q2: How do you prevent the model from learning "long contract = vulnerable"?**

Complexity-proxy risk detection in Stage 6 analysis. If the `feature_dist` module finds any class where positives are systematically larger/longer than negatives, it flags a HIGH-RISK pair in `complexity_proxy_risk.md`. The model team then adds LoC as a control feature or applies per-contract importance weighting.

**Q3: What's the most likely failure mode for Run 11?**

Slither API change (the Run 9 failure). The freshness checker monitors this, but it's reactive — it flags stale Slither AFTER the pipeline starts. A proactive fix: pin Slither to a known-good version in the Docker image and verify compatibility in CI before the pipeline runs.

**Q4: Why 10 classes? Why not 25 or 5?**

The 9+1 taxonomy (9 vulnerability classes + NonVulnerable) is the intersection of DASP, SWC, and CWE classifications that all 6 critical-path sources can map to. 25 classes would leave most classes under a source's labeling threshold. 5 classes would lose the granularity needed for smart contract audit tools.

**Q5: How do you handle multi-label contracts?**

Two mechanisms: (1) DIVE's folder-membership approach naturally produces multi-label contracts (same file in multiple folders). (2) The merger reconciles across sources — if SolidiFI says Reentrancy and DIVE says AccessControl, the merged label has both. Multi-label contracts are not excluded; they're counted via `multi_label_count` in balance_viz.

**Q6: What's the minimum viable corpus threshold?**

4000 total contracts, 300 per major class (Reentrancy, DoS, IntegerUO), 100 per minor class, 300 for CallToUnknown, and 90% recall on SmartBugs Curated's 143 ground-truth contracts. If these aren't met, the pipeline stops with "NOT VIABLE" at the gate.

**Q7: Why is the 3:1 NonVulnerable cap important?**

Without it, the model sees 428 NonVulnerable contracts for every 1 vulnerable one. The model learns "predict NonVulnerable and be right 99.8% of the time" — the BCCC failure pattern. The 3:1 cap forces the model to actually learn vulnerability patterns.

**Q8: What happens if the pre processing manifest is missing a contract?**

The `ingestion_manifest.json` is the hard gate between Stages 1 and 2. If a contract is missing from the manifest, the graph extraction stage doesn't know about it — it's excluded from the pipeline. The label parsers skip it too (they read from preprocessed directories, which are also gated by the manifest).

**Q9: How does the deduplicator handle identical contracts with different names?**

Level 1 (SHA-256) catches byte-identical files regardless of name. Level 3 (normalized-text) catches contracts that differ only in comments or whitespace. Together, these handle the "same contract, different filename" case across sources.

**Q10: Why Slither as the intermediary for graph extraction instead of a direct Solidity parser?**

Slither's intermediate representation (SlithIR) handles 95% of Solidity constructs out of the box — including the edge cases that break standalone parsers (e.g., `abi.encodeWithSelector`, inline assembly, yul). The cost is Slither API dependency (the Run 9 failure), but the alternative is reimplementing a Solidity parser from scratch, which has its own failure modes.
