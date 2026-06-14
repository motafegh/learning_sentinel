# R07 — Recall: Sessions 23–28 (Verification + Splitting + Label Parsers + Merger + Export Overview)

**Date:** 2026-06-14
**Session number:** R07 of 08
**Purpose:** Consolidate Sessions 23–28 before interview or code review
**Estimated study time:** 20 min
**Status:** 🟢 Complete

---

## §1 Modules Covered

| Session | Module | LOC |
|---|---|---|
| 23 | Verification Runner + Gate | slither_runner 250, tool_validator 273, gate 286 |
| 24 | Semantic Checker + FP Estimator + Negative Checker + Probe | 283 + 336 + 235 + 587 |
| 25 | Splitting | splitters 441, nonvulnerable_cap 163, dedup_enforcer 116, leakage_auditor 163 |
| 26 | Label Parsers | dive 167, smartbugs_curated 165, solidifi 162 |
| 27 | Label Merger | merger 240 |
| 28 | Export Overview + Chunker | chunker 237, export 173 |

**Total LOC covered:** ~3,795

---

## §2 Key Decisions Recap

### Verification (23–24)
- **Verdict hierarchy:** VERIFIED > PROVISIONAL > BEST-EFFORT > FAIL
- **Gate pass requires:** no FAIL classes, or FAIL classes are optional
- **FP_FAIL_THRESHOLD = 30%** — if Slither fires on negatives for a class >30%, the class gets FAIL
- **Semantic checker** uses graph features (CEI path, block globals, unchecked blocks) — not source code
- **Probe trivials** detects "too easy" positives — high rate means the class is well-covered but model may overfit

### Splitting (25)
- **Two-pass:** splitter (4 strategies) → dedup_enforcer (reassign near-dup groups)
- **NonVulnerable cap:** 3:1 max, stratified by source, applied AFTER splitting
- **Leakage auditor:** independent text-shingle (Jaccard, threshold 0.5), informational only
- **Project-level splitter:** entire DApps kept in one split ("defihacklabs: project")

### Labeling (26–27)
- **Parsers share a pattern:** `original_path` → folder → crosswalk YAML → canonical class
- **DIVE is the only multi-label parser** — same `.sol` file in multiple vulnerability folders
- **Merger tier precedence:** T0 > T1 > T2 > T3 > T4, positive wins within same tier
- **Co-occurrence flagging:** T3/T4 sources with DoS+Reentrancy >50% flagged per contract

### Export (28)
- **Fix A:** manifest.json written LAST, excluded from artifact hash
- **Export layout:** 2 parquet files + 2 sharded .pt directories
- **SentinelDatasetExport** is the bridge to ML side (read-only, hash verification)

---

## §3 Common Misconceptions

| Misconception | Correction |
|---|---|
| The gate REMOVES FAIL contracts | The gate BLOCKS FAIL classes from export. Contracts still exist in labels. |
| The dedup enforcer catches EVERY leak | No — it only catches dedup groups from preprocessing. The leakage auditor is the safety net. |
| DIVE's "Bad Randomness" contracts are dropped entirely | They're written as NonVulnerable (conservative fallback) |
| The NonVulnerable cap applies to the whole corpus | It applies per-split, independently (train/val/test each capped) |
| The merger overwrites conflicting labels | It merges with tier precedence — positive from highest tier wins |
| export.py IS the chunker | export.py is the CONSUMER API; chunker.py is the orchestrator |

---

## §4 Quick-Reference: Thresholds

| Threshold | Value | Effect |
|---|---|---|
| `FP_RATE_FAIL_THRESHOLD` | 30% | fp_estimator → FAIL |
| `CO_OCCUR_NOISE_THRESHOLD` | 50% | Merger flags DoS+Reentrancy |
| `ast_similarity_threshold` | 0.85 | Dedup + enforcer + overlap |
| `NonVulnerable cap` | 3:1 | negative.positive_ratio_max |
| `text_similarity_threshold` | 0.5 | Leakage auditor Jaccard |
| `SHINGLE_SIZE` | 3 | Leakage auditor n-grams |

---

## §5 Parser Path Trick — Interview Shortcut

The 3 label parsers differ only in how they extract the folder from `original_path`:

```
SolidiFI:    repo/buggy_contracts/<FOLDER>/buggy_N.sol  → parts[2]
SmartBugs:   repo/<FOLDER>/<contract>.sol                → parts[1]
DIVE:        No folder from path — uses _build_folder_index()
```

If asked "how would you add a new parser?", answer: write `_extract_folder()` to parse the source's path convention, write a crosswalk YAML, and follow the existing `_build_labels_json()` template.

---

## §6 File Paths to Memorize

```
data/labels/<source>/<sha256>.labels.json       # per-source labels
data/labels/merged/<sha256>.labels.json          # merged labels
data/splits/<run_id>/{train,val,test}.jsonl      # split JSONLs
data/exports/<version>/manifest.json             # export artifact
data/exports/<version>/labels.parquet             # training-ready labels
data/analysis/<run_id>/verification_report.md     # gate report
```

---

## §7 Interview Challenge

**Question:** "You add a new source with 1,000 contracts. 600 are Reentrancy (T3). 200 are NonVulnerable. 200 are AccessControl (T2). Walk through what happens."

**Answer:**
1. **Ingest + preprocess** → `data/preprocessed/<source>/` with meta.json per contract
2. **Label parser** reads `original_path`, maps folder → crosswalk → canonical class. 600 Reentrancy (T3), 200 AccessControl (T2), 200 NonVulnerable
3. **Merger** reconciles with other sources. For contracts already labeled by SolidiFI (T0), T0 wins → Reentrancy=T0. For new contracts, T3 or T2 stands
4. **Class auditor** checks per-class counts, co-occurrence
5. **Semantic checker** verifies Reentrancy labels against graph CEI paths, computes per-class pass_rate
6. **FP estimator** tests Slither detectors on the 200 AccessControl negatives → if Slither fires on >30% → FP_FAIL
7. **Gate**: Reentrancy ≥ VERIFIED if pass_rate >0.80, fp_rate <0.30, tool_agreement >0.80
8. **Splitter**: stratified by class + tier. Dedup enforcer checks if any new contract shares a dedup_group with existing contracts → keeps same split
9. **Leakage auditor**: text-shingle check between new and existing contracts → report
10. **NonVulnerable cap**: if total NonVulnerable > 3× positives → subsample by source
11. **Export**: included in next version's shards, parquet updated, new artifact_hash
