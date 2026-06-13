# Reference — `sentinel-data` (Data Module v2)

> **This file is the entry point for learning the `sentinel-data` Python package.**
> Read this first, then `preferences.md`, then start at Session 1 of the roadmap.

**Source-of-truth rule:** this file describes what is **currently in the code at
`~/projects/sentinel/data_module/`**. Every fact was verified against the
package as of **2026-06-12** (the v2-readiness report date). If the code
changes, update this file.

---

## 1. What This Module Is

`sentinel-data` is the **data engineering layer** of SENTINEL. It turns raw
Solidity source code from 17 curated corpora into model-ready training
artifacts, then hands them to `sentinel-ml` via a versioned export seam.

- **Package:** `sentinel-data` v0.1.0
- **Source tree:** `~/projects/sentinel/data_module/`
- **Entry point:** `sentinel-data` CLI → `sentinel_data.cli:main`
- **Python:** 3.12 (locked)
- **Build system:** Poetry with 2 optional groups — `pipeline` (heavy solidity
  tooling: slither, solc-select, dvc, huggingface_hub) and `ml` (torch, torch-geometric, transformers)
- **Total LOC in `sentinel_data/`:** 12,478 (Python only; 49 files)
- **One-way dependency:** `sentinel-data` NEVER imports from `sentinel-ml`.
  Training consumes artifacts produced here; training code never feeds back.

---

## 2. Pipeline (9 stages + 2 utilities)

Per `sentinel_data/cli.py:71-93` and `dvc.yaml`. The pipeline is a strict
left-to-right DAG. Each stage reads the previous stage's well-known output dir
under `data/`.

| # | Stage | CLI subcommand | Module path | Status |
|---|-------|----------------|-------------|--------|
| 1 | ingest (1a) | `sentinel-data ingest` | `sentinel_data/ingestion/` | ✅ |
| 2 | preprocess (1b) | `sentinel-data preprocess` | `sentinel_data/preprocessing/` | ✅ |
| 3 | represent (2) | `sentinel-data represent` | `sentinel_data/representation/` | ✅ |
| 4 | label (3) | `sentinel-data label` | `sentinel_data/labeling/` | ⚠️ CLI STUB; merger runs from Python |
| 5 | verify (4) | `sentinel-data verify` | `sentinel_data/verification/` | ✅ |
| 6 | split (5a) | `sentinel-data split` | `sentinel_data/splitting/` | ✅ |
| 7 | register (5b) | `sentinel-data register` | `sentinel_data/registry/` | ✅ |
| 8 | analyze (6) | `sentinel-data analyze` | `sentinel_data/analysis/` | ✅ |
| 9 | export (7) | `sentinel-data export` | `sentinel_data/export/` | ⏳ STUB (10-line `__init__.py` only) |
| — | (utility) | `sentinel-data freshness` | `sentinel_data/ingestion/freshness.py` | ✅ |
| — | (orchestrator) | `sentinel-data run [--from-stage STAGE]` | dispatch table `cli.py:679-690` | ✅ |

**Data flow** (per `docs/architecture.md`):

```
raw ─► ingest ─► preprocessed ─► preprocess ─► representations ─► label ─►
verification ─► split ─► splits ─► register ─► registry ─► analyze ─► exports
```

**Per-stage contracts** (input → output):

| Stage | Input | Output | Sidecar |
|-------|-------|--------|---------|
| ingest | `config.yaml` enabled sources | `data/raw/<source>/*.sol` | `ingestion_manifest.json` (SHA-256) |
| preprocess | `data/raw/*.sol` | `data/preprocessed/<source>/<sha256>.sol` | `<sha256>.meta.json` |
| represent | `<sha256>.sol` + `<sha256>.meta.json` | `data/representations/<source>/<sha256>.pt` + `.tokens.pt` | `<sha256>.rep.json` |
| label | graphs + 17 crosswalk YAMLs | `data/labels/multilabel_index.csv` | co-occurrence matrix |
| verify | labels + graph features | `data/verification/contracts_clean.csv` | `verification_report_<ts>.md` |
| split | `contracts_clean.csv` | `data/splits/{train,val,test}.csv` | `split_manifest.json` |
| register | split CSVs | `data/registry/catalog.sqlite` | YAML mirror |
| analyze | registry + labels | `data/analysis/complexity_proxy_risk.md` + co-occurrence CSVs | — |
| export | registry + representations | `data/exports/*.shard.tar` | manifest |

---

## 3. File Inventory — What We Will Learn

LOC from `wc -l` on `find sentinel_data -name "*.py"`. **Bold = unconditional
chunking per global P3 (complex / large / important).**

### 3.1 Top-level

| File | LOC | Role | Teaching approach |
|------|-----|------|-------------------|
| **`cli.py`** | **1037** | Entry point: 9 subcommands + `run` orchestrator + `sys.path` bootstrap (lines 62–68) | Chunked (≈3 chunks: bootstrap+dispatch / per-stage handlers / freshness+run) |
| `__init__.py` | 57 | Package docstring + `__version__ = "0.1.0"` | Read in 1 pass |

### 3.2 `sentinel_data/ingestion/` — Stage 1a (1,184 LOC)

| File | LOC | Role |
|------|-----|------|
| `ingest.py` | 113 | Orchestrator: `ingest_source()` / `ingest_all()` |
| `manifest.py` | 111 | `IngestionManifest` + `FileRecord` + `verify_manifest` (SHA-256) |
| `freshness.py` | 120 | Pin staleness + `slither-analyzer` version check → `data/analysis/freshness_report.md` |
| `label_folderize.py` | 173 | Per-class symlinks for CSV-labeled sources (DIVE) |
| `connectors/base.py` | 95 | Connector interface |
| `connectors/git_connector.py` | (read on demand) | git clone + pin checkout |
| `connectors/manual_connector.py` | 189 | ZIP extract + macOS `__MACOSX/` strip |
| `connectors/etherscan_connector.py` | (read on demand) | NEW connector for DISL (514K unlabeled) |
| `connectors/huggingface_connector.py` | (read on demand) | HF dataset pull |
| `connectors/zenodo_connector.py` | (read on demand) | Zenodo record download |

### 3.3 `sentinel_data/preprocessing/` — Stage 1b (1,512 LOC)

| File | LOC | Role |
|------|-----|------|
| `pipeline.py` | 246 | `PreprocessingPipeline` + `ContractMeta` sidecar |
| `preprocess.py` | 264 | CLI service: `--sample N`, `--retry-failed` |
| **`flattener.py`** | **311** | solc `--flatten` + 2-stage unresolved-import strip fallback |
| `compiler.py` | 191 | 2-pass: exact pragma → nearest available solc |
| `deduplicator.py` | (read on demand) | 3-level: SHA-256 → address → AST near-dup (stub) |
| `normalizer.py` | (read on demand) | Strip comments, SPDX, whitespace |
| `segmenter.py` | (read on demand) | Version bucket + `has_unchecked_block` flag |
| `parallel.py` | 156 | Multiprocessing wrapper |
| `_transitive_strip.py` | 104 | Helper for flattener's import-strip fallback |

### 3.4 `sentinel_data/representation/` — Stage 2 (1,302 LOC) ⚠ MOST CODE LIVES IN `ml/src/`

| File | LOC | Role |
|------|-----|------|
| `__init__.py` | 64 | Re-exports public symbols |
| `graph_schema.py` | 148 | **Thin adapter** → `ml/src/preprocessing/graph_schema.py` (v9 schema source of truth) |
| `graph_extractor.py` | 77 | **Thin adapter** → `ml/src/preprocessing/graph_extractor.py` |
| `tokenizer.py` | 72 | **Thin adapter** → `ml/src/data_extraction/windowed_tokenizer.py` (NOT `tokenizer.py` — single-window codebert-base, wrong) |
| **`orchestrator.py`** | **344** | v2 manifest-driven: per-contract cache check, solc resolve, graph+token extract |
| `cache_manager.py` | 119 | Content-addressed cache: key = `(sha256, schema_version, extractor_version)` |
| `versioner.py` | 100 | Schema/extractor registry at `data/representations/_version_registry.json` |
| **`cfg_builder.py`** | **309** | Opt-in standalone CFG (`--emit-cfg`); consumed by Stage 6 `complexity_proxy_risk` |
| `call_graph.py` | 35 | **DEFERRED v3.1** — placeholder |
| `opcode_extractor.py` | 36 | **DEFERRED v3.1** — placeholder |
| `pdg_builder.py` | 34 | **DEFERRED v3.1** — placeholder |

### 3.5 `sentinel_data/labeling/` — Stage 3 (~700 LOC)

| File | LOC | Role |
|------|-----|------|
| `merger.py` | 240 | Multi-source merger; tier precedence; **99% DoS↔Reentrancy co-occurrence flag** |
| `gate.py` | 139 | Minimum-viable-corpus Go/No-Go gate |
| `parsers/dive.py` | 167 | DIVE multi-label parser (22,330 contracts) |
| `parsers/solidifi.py` | 162 | SolidiFI parser (9,369 injected bugs) |
| `schema/taxonomy.yaml` | (read) | Canonical 10-class taxonomy (labeling order) |
| `crosswalks/{dive,solidifi,defihacklabs,smartbugs_curated,...}.yaml` | (read on demand) | Per-source class maps |

### 3.6 `sentinel_data/verification/` — Stage 4 (2,709 LOC) — LARGEST SUBPACKAGE

| File | LOC | Role |
|------|-----|------|
| `class_auditor.py` | 189 | 10×10 co-occurrence matrix; flag at `P(B\|A) > 0.50` |
| `semantic_checker.py` | 283 | Per-class graph-feature checks; 3/10 classes = `NOT_EXTRACTABLE` |
| `tool_validator.py` | 273 | Slither per-class agreement |
| `fp_estimator.py` | 336 | Stratified-by-(source, tier) FP rate |
| `negative_checker.py` | 235 | NonVulnerable Slither hit rate; warn 5% / fail 10% |
| `gate.py` | 286 | `VERIFIED` / `PROVISIONAL` / `BEST-EFFORT` / `FAIL` per class |
| `report_generator.py` | 272 | 9-section markdown report |
| `slither_runner.py` | 250 | Shared Slither runner; `CLASS_TO_DETECTORS` + content-addressed cache |
| `probe_dataset.py` | 361 | 40 real + 1 trivial pos + 1 trivial neg per class |
| `probe_trivials.py` | 226 | 10 hand-crafted trivial positives + 1 trivial negative |
| `patterns/*.yaml` | 10 files | Per-class pattern specs (documentation only — semantic_checker does NOT read them) |

### 3.7 `sentinel_data/splitting/` — Stage 5a (883 LOC)

| File | LOC | Role |
|------|-----|------|
| **`splitters.py`** | **441** | `Contract` / `Splits` / `SplitMetadata` + 4 splitter functions |
| `dedup_enforcer.py` | 116 | BCCC-failure-pattern fix (post-split dedup) |
| `leakage_auditor.py` | 163 | Post-split text shingle safety net |
| `nonvulnerable_cap.py` | 163 | 3:1 NonVulnerable cap (friend review 2026-06-09) |

### 3.8 `sentinel_data/registry/` — Stage 5b (799 LOC)

| File | LOC | Role |
|------|-----|------|
| **`catalog.py`** | **541** | SQLite + YAML mirror; 4 base + 2 system tables |
| `dataset_diff.py` | 161 | Per-class metric projection |
| `lineage_tracker.py` | 98 | DAG of transformations + `hash_artifact` / `verify_artifact` |

### 3.9 `sentinel_data/analysis/` — Stage 6 (1,486 LOC)

| File | LOC | Role |
|------|-----|------|
| `balance_viz.py` | 134 | Per-class / per-source / per-tier counts |
| **`feature_dist.py`** | **436** | ⭐ The "Run 9 failure catcher" → `complexity_proxy_risk.md` |
| `cooccurrence.py` | 187 | Directed + conditional matrices |
| `overlap_detector.py` | 267 | Pairwise Jaccard between source datasets |
| `drift_monitor.py` | 298 | KS test for feature + label distribution |
| `probe_dataset.py` | (re-export) | → `verification/probe_dataset.py` |

### 3.10 `sentinel_data/export/` — Stage 7 (STUB)

| File | LOC | Role |
|------|-----|------|
| `__init__.py` | 10 | Module docstring only |
| `chunker.py` | 210 | STUB scaffolding |
| `export.py` | 141 | STUB scaffolding |
| `graph_writer.py` | 102 | STUB scaffolding |
| `label_writer.py` | 150 | STUB scaffolding |
| `metadata_writer.py` | 200 | STUB scaffolding |
| `token_writer.py` | (read) | STUB scaffolding |
| `format_schema/v1.yaml` | (read) | Planned format schema |

> All Stage 7 code is in place but the seams are not yet wired to the v2 export
> flow. The actual export that worked is in `ml/_archive/seam_swap_pre_2026-06-12/`.

---

## 4. Tests

**Location:** `data_module/tests/` (≈94 tests + 2 integration, per `data_module/README.md:130`)

| Test dir | File count | Coverage |
|----------|-----------|----------|
| `test_skeleton.py` | 1 | Stage 0 smoke |
| `test_integration_solidifi.py` | 1 | End-to-end on SolidiFI |
| `test_integration_dive.py` | 1 | End-to-end on DIVE |
| `test_ingestion/` | 3 | Connectors, manifest, label_folderize |
| `test_preprocessing/` | 2 | Pipeline + retry-failed |
| **`test_representation/`** | **5** | **Thin-adapter + byte-identical + 13-issue preservation + SolidiFI A1/A2/A3 + EMITS fixture** |
| `test_labeling/` | 7 | Crosswalks, gate, merger, parsers, taxonomy |
| **`test_verification/`** | **12** | **BCCC regression + all 10 checkers + patterns + report + CLI** |
| `test_splitting/` | 1 | Splitter correctness |
| `test_registry/` | 1 | Catalog |
| `test_analysis/` | 1 | Feature dist |
| `test_export/` | 5 | Stub/format tests |

**Run command:** `cd ~/projects/sentinel/data_module && poetry run pytest tests/ -v`

**The 5 critical tests (per `docs/architecture.md`):**
1. Byte-identical regression (`test_representation/test_byte_identical_regression.py`) — merge gate for extractor changes
2. BCCC Phase 5 regression (`test_verification/test_bccc_regression.py`) — merge gate for verification changes
3. Stage 2 regression (`test_representation/test_13_issue_preservation.py`) — 13 bug-fix regressions
4. Schema-dim gate (`x.shape[-1] == NODE_FEATURE_DIM == 12`) — prevents Run 8 v8/v9 silent mix
5. SentinelDataset round-trip (`ml/tests/test_sentinel_dataset.py`, 16 tests) — end-to-end on v2 export

---

## 5. Current Status (2026-06-12, v2-readiness report)

**Verdict:** 6/7 GREEN, 1/7 PARTIAL (Gate 5 corpus-bound, not code-bound).

| Gate | Verdict | Notes |
|------|---------|-------|
| 1. Schema regression (Stage 2 byte-identical) | ✅ | 40/40 tests |
| 2. BCCC Phase 5 verification suite | ✅ | 191 pass / 21 skip (solc/external data) |
| 3. End-to-end round-trip (SentinelDataset) | ✅ | 16/16 + ad-hoc smoke (15,063 train + 3,226 val) |
| 4. Feature distribution (Stage 6) | ✅ | By construction — v9 schema unchanged by seam swap |
| 5. All 10 classes VERIFIED or PROVISIONAL | 🟡 | 2 VERIFIED, 5 PROVISIONAL, **3 BEST-EFFORT** (corpus-limited: no SmartBugs Curated yet) |
| 6. No leakage across splits | ✅ | 0 overlap (15,644 / 3,344 / 3,368) |
| 7. No open code-bug regression | ✅ | EMITS (BUG-H7) + predictor (F8/F10) both fixed |

**Per-class gate verdicts (2026-06-11 corpus, SolidiFI+DIVE, 22,356 contracts):**

| Class | Verdict | Notes |
|-------|---------|-------|
| Reentrancy | ⚠️ PROVISIONAL | 67% semantic pass, co-occur with DoS |
| CallToUnknown | 🔶 BEST-EFFORT | Only 39 contracts (SolidiFI only) |
| Timestamp | ⚠️ PROVISIONAL | 83% semantic pass, co-occur |
| ExternalBug | 🔶 BEST-EFFORT | 0% coverage on T0 (no probe representations) |
| GasException | ⚠️ PROVISIONAL | 0% coverage on T0 |
| DenialOfService | 🔶 BEST-EFFORT | Co-occur with Reentrancy 70.8% |
| IntegerUO | ⚠️ PROVISIONAL | 100% semantic pass, co-occur |
| UnusedReturn | ⚠️ PROVISIONAL | 100% semantic pass, co-occur |
| MishandledException | ✅ VERIFIED | 100% semantic pass, 97% coverage |
| TransactionOrderDependence | ✅ VERIFIED | 100% coverage, co-occur |

**Distribution:** 2 VERIFIED, 5 PROVISIONAL, 3 BEST-EFFORT, **0 FAIL**.

**Run 11 launch:** 2026-08-18 (per `data_module/README.md:179`).

---

## 6. Critical Concepts (must internalize early)

### 6.1 The Two-Taxonomy Divergence ⚠️ MOST IMPORTANT

There are **two `class_names()` orders in the codebase** that are NOT the same:

- **Representation order** (`sentinel_data/representation/graph_schema.py:73-84`):
  Reentrancy=0, CallToUnknown=1, Timestamp=2, ExternalBug=3, GasException=4,
  DoS=5, IntegerUO=6, UnusedReturn=7, MishandledException=8, **NonVulnerable=9**.
  9 vulnerability classes + NonVulnerable. **This is what the model checkpoint uses.**
- **Labeling order** (`sentinel_data/labeling/schema/taxonomy.yaml`):
  CallToUnknown=0, DoS=1, ExternalBug=2, GasException=3, IntegerUO=4,
  MishandledException=5, Reentrancy=6, Timestamp=7,
  **TransactionOrderDependence=8, UnusedReturn=9**. 9 vulnerability classes +
  UnusedReturn. NonVulnerable is a *negative* label, not a slot.

**Rule:** anything that depends on **index alignment** with the existing model
checkpoints must use the **representation order**. String-keyed lookups work
either way. Source: `data_module/README.md:218-223`.

### 6.2 The Thin-Adapter Pattern

`sentinel_data/representation/{graph_schema,graph_extractor,tokenizer}.py` are
**~10-line re-export wrappers** over `ml/src/preprocessing/*.py` and
`ml/src/data_extraction/windowed_tokenizer.py`. The `is` equality test in
`test_representation/test_thin_adapter.py` enforces that
`from sentinel_data.X import Y` returns the same object as `from ml.X import Y`.

**Bug history:** `tokenizer.py` originally pointed at `ml/src/data_extraction/tokenizer.py`
(single-window codebert-base, wrong) — fixed to point at `windowed_tokenizer.py`
(graphcodebert-base, `[4,512]` sliding window). See
`sentinel_data/representation/README.md:90` and `tokenizer.py:11-15`.

**Stage 7 plan:** delete the wrappers, rebind the import. The seam-swap
already done in `ml/src/datasets/sentinel_dataset.py` is the proof.

### 6.3 The v9 Schema (LOCKED)

| Constant | Value | Source line |
|----------|-------|-------------|
| `FEATURE_SCHEMA_VERSION` | `"v9"` | `ml/src/preprocessing/graph_schema.py:161,175,218` (re-exported) |
| `NODE_FEATURE_DIM` | `12` | per-node feature vector |
| `NUM_NODE_TYPES` | `14` | includes `CFG_NODE_ARITH=13` (added in v9) |
| `NUM_EDGE_TYPES` | `12` | includes `EXTERNAL_CALL=11` (added in v9) |
| `_MAX_TYPE_ID` | `13.0` | derived from `max(NODE_TYPES.values())` |
| `NUM_CLASSES` | `10` | LOCKED to match existing checkpoints |
| `EXTRACTOR_VERSION` | `"v2.1-windowed-gcb"` | bumped when windowed tokenizer was added |

**To change the schema:** bump `FEATURE_SCHEMA_VERSION` + update
`graph_schema.py:73-84` AND `labeling/schema/taxonomy.yaml:21-159` AND
`data/processed/multilabel_index.csv` (the v1 column-order contract).

### 6.4 Content-Addressed Cache

Cache key = `(sha256_of_source, schema_version, extractor_version)`. Change
any of the three → cache miss for that file. Cache lives in
`data/representations/<source>/` as 3 files per contract (`.pt`, `.tokens.pt`, `.rep.json`).

**Version registry:** `data/representations/_version_registry.json` tracks
the active `(schema_version, extractor_version)`. `versioner.py:check_and_evict()`
(66) evicts stale cache entries on registry mismatch — **this is the fix for
the Run 8 v8-vs-v9 silent mix** (see `representation/README.md:135`).

### 6.5 The 99% Co-occurrence Rule (the BCCC-failure signal)

`merger.py:100-124` flags any (class_a, class_b) pair where
`P(class_b=1 | class_a=1) > 0.50`. BCCC had DoS↔Reentrancy at 99%; the
threshold of 50% is conservative.

`class_auditor.py:24-26`: `CO_OCCUR_FLAG_THRESHOLD = 0.50` (very conservative;
BCCC was 99%).

### 6.6 The 4-Layer Verification Defense

| Layer | File | Speed | Role |
|-------|------|-------|------|
| 1. Semantic checker | `semantic_checker.py` | Fast (no Slither) | Graph-feature per-class checks; 3/10 classes = `NOT_EXTRACTABLE` |
| 2. Tool validator | `tool_validator.py` | Slow | Slither per-class agreement |
| 3. FP estimator | `fp_estimator.py` | Slow | Stratified (source × tier) FP rate |
| 4. Negative checker | `negative_checker.py` | Slow | Slither hit rate on NonVulnerable |

**IntegerUO has NO Slither detector in v0.10** (Solidity 0.8+ handles it
natively). For IntegerUO, only the semantic checker can corroborate. Source:
`slither_runner.py:26-45` + `verification/README.md:63`.

### 6.7 Friend-Review Rules (added 2026-06-09 in `config.yaml`)

1. **NonVulnerable 3:1 cap** (`pipeline.negative.positive_ratio_max: 3.0`):
   NonVulnerable count ≤ 3× total positive count across all 10 classes.
   Enforced in Stage 5a stratified_splitter.
2. **CallToUnknown merge rule** (`pipeline.class.merge_rules`): if
   `CallToUnknown_verified_count < 300` → `pause_and_ask_human` (not silent
   auto-merge). Reversible.
3. **Minimum-viable-corpus gate** (`pipeline.min_viable_corpus`):
   - `total_contracts_min: 4000`
   - `per_class_positive_min_major: 300` (Reentrancy, DoS, IntegerUO)
   - `per_class_positive_min_minor: 100` (other 7)
   - `call_to_unknown_min: 300`
   - `smartbugs_curated_recall_min: 0.90`
   - `forge_agreement_min: 0.85`
4. **DIVE "bad randomness" dropped** (no 10-class equivalent).

### 6.8 `sys.path` Bootstrap

`cli.py:62-68` adds the SENTINEL repo root and `ml/` to `sys.path` before any
imports. This is the **only** place in production code where `sys.path` is
manipulated. Tests use `conftest.py` instead. The two strategies are
intentionally parallel: the CLI must work without pytest, the test suite must
work without the CLI.

---

## 7. Deferred Items / Known Gaps

### 7.1 Deferred to v2.1 (per `config.yaml:219-363` + `README.md:181-199`)

- **DeFiHackLabs** — Foundry project; 715/738 dropped because forge-std
  cheatcodes can't be compiled by standalone solc. Re-enable when git
  connector supports forge-std clone. (`config.yaml:108-120`)
- **BCCC** — 89.4% Reentrancy FP, 86.9% CallToUnknown FP (per Phase 5
  verification). Legacy outputs at `docs/legacy/bccc_deep_dive/`. v1.4
  verified labels (24,021 contracts) may be re-introduced as gold supplement
  in v2.1.
- **12 additive sources** (Bastet, FORGE, ScrawlD, DeFi Hacks REKT, Ethernaut,
  OpenZeppelin Contracts, solidity_defi_vulns, DeFiVulnLabs, SC-Bench, SmartBugs
  Wild, slither-audited, Zenodo CLEAR, Messi-Q, ScaBench) — only ingested if
  Stage 3 finishes early.

### 7.2 Dropped

- **ReentrancyStudy** (230K single-class) — recreates BCCC imbalance
- **Code4rena scraper** — replaced by Bastet (legal risk)

### 7.3 Stage 7 STUB

`sentinel_data/export/` has scaffolding code (chunker, writers, format_schema)
but the `__init__.py` is 10 lines and the seam is not yet wired. The
seam-swap already done in `ml/src/datasets/sentinel_dataset.py` (16 tests
pass) is the working end-to-end proof.

### 7.4 Stage 3 CLI STUB

`cli.py:223-229` is a stub. The merger runs from Python today.

### 7.5 Representation v3.1 Stubs

`call_graph.py`, `opcode_extractor.py`, `pdg_builder.py` — all 35-line
`NotImplementedError` placeholders. The ICFG-Lite (CALL_ENTRY/RETURN_TO
edges in the main graph) already encodes the most important cross-function
topology.

### 7.6 Open Code Bugs (per ADR-0002 + v2-readiness)

| Bug | Status | Location |
|-----|--------|----------|
| EMITS edge (BUG-H7) | ✅ FIXED | `graph_extractor.py:1653-1940` (two-path emit detection) |
| Predictor tier threshold (F8/F10) | ✅ FIXED | `ml/src/inference/predictor.py:707-718` (per-class thresholds) |
| CALL_ENTRY cross-function external calls | 🟡 OPEN | `ml/src/preprocessing/graph_extractor.py:1001` — partial fix; full fix post-Run-11 |
| 22+ pre-existing test failures | 🟡 OPEN | `ml/tests/test_preprocessing.py::TestExtractionIntegration` — v8→v9 schema test drift + Slither API changes. Out of scope for Stage 7B. |

---

## 8. Spec File Update Protocol (per global reference.md)

When something changes in this module, update the spec files **in this order**:

| Trigger | File(s) to update | Timing |
|---------|-------------------|--------|
| Code changes (`sentinel_data/**` or thin-adapter targets) | `reference.md` §3, §5, §7 | Same session |
| A new module-specific learning rule | `preferences.md` | Immediately |
| An `[AUDIT]` flag raised | `audit_flags.md` | Same response |
| A chunk finishes | `session_log.md` | End of that response |
| Run 11 ships (changes to schema, sources, gate verdicts) | `reference.md` §5, §6, §7 | Same session |

**Do NOT delete entries** from `audit_flags.md` or `session_log.md`. Append only.

---

## 9. Roadmap — Suggested Learning Order

> Per global P3: complex / large / important files get chunked. Per global
> P5: every new file opens with big picture. Per global P10: warm-up recall
> + 3-things-to-lock-in at the end of each chunk.

| Session | Chunk | File(s) | Est. time | Why this order |
|---------|-------|---------|-----------|----------------|
| 1 | Orientation | `data_module/README.md` + `config.yaml` + `docs/architecture.md` + `dvc.yaml` + `pyproject.toml` | 30 min | Big picture before any code |
| 2 | Entry point | `sentinel_data/__init__.py` + `cli.py:1-200` (sys.path + STAGES table + dispatch) | 45 min | See the whole pipeline from the top |
| 3 | CLI handlers | `cli.py:200-700` (per-stage handlers) | 1 hr | Understand what each stage does at the CLI surface |
| 4 | CLI orchestrator | `cli.py:700-1037` (`run` + `freshness` subcommands) | 30 min | Pipeline runner + utility |
| 5 | Ingestion overview | `ingestion/ingest.py` + `manifest.py` + `freshness.py` | 1 hr | Stage 1a first (before preprocessing) |
| 6 | Ingestion connectors | `connectors/base.py` + `git_connector.py` + `manual_connector.py` | 1 hr | Strategy pattern |
| 7 | Ingestion label folderize | `ingestion/label_folderize.py` (173 LOC) | 45 min | DIVE-specific symlink logic |
| 8 | Preprocessing overview | `preprocessing/pipeline.py` + `preprocess.py` | 1 hr | The pipeline coordinator |
| 9 | Preprocessing: flattener | `preprocessing/flattener.py` (311 LOC) | 1.5 hr | Unconditional chunking — complex |
| 10 | Preprocessing: compiler + dedup | `preprocessing/compiler.py` (191) + `deduplicator.py` | 1 hr | 2-pass compile + 3-level dedup |
| 11 | Preprocessing: rest | `normalizer.py` + `segmenter.py` + `parallel.py` + `_transitive_strip.py` | 1 hr | Smaller files; one pass each |
| 12 | Representation: thin-adapter | `representation/graph_schema.py` + `graph_extractor.py` + `tokenizer.py` + `__init__.py` | 45 min | ⭐ 4 thin wrappers, ~300 LOC total — must read **the actual `ml/src/` files** to learn the real logic |
| 13 | Representation: orchestrator | `representation/orchestrator.py` (344 LOC) | 1.5 hr | v2 manifest-driven; per-contract flow |
| 14 | Representation: cache + versioner + CFG | `representation/cache_manager.py` + `versioner.py` + `cfg_builder.py` | 1.5 hr | The "Run 8 v8/v9 silent mix" fix |
| 15 | Representation: v3.1 stubs | `call_graph.py` + `opcode_extractor.py` + `pdg_builder.py` | 15 min | Awareness only — they `raise NotImplementedError` |
| 16 | Labeling: schema + parsers | `labeling/schema/taxonomy.yaml` + `parsers/dive.py` + `parsers/solidifi.py` | 1.5 hr | The 10-class taxonomy + the 2 active parsers |
| 17 | Labeling: merger + gate | `labeling/merger.py` (240) + `gate.py` (139) | 1.5 hr | Tier precedence + 99% co-occurrence flag + Go/No-Go |
| 18 | Labeling: crosswalks | `labeling/crosswalks/*.yaml` (5 active + stubs) | 30 min | Per-source class maps |
| 19 | Verification: class_auditor | `verification/class_auditor.py` (189) | 1 hr | The co-occurrence matrix — the BCCC-failure signal |
| 20 | Verification: semantic_checker | `verification/semantic_checker.py` (283) | 1.5 hr | Per-class graph-feature checks; 3 NOT_EXTRACTABLE |
| 21 | Verification: slither_runner | `verification/slither_runner.py` (250) | 1.5 hr | `CLASS_TO_DETECTORS` + content-addressed cache |
| 22 | Verification: tool_validator | `verification/tool_validator.py` (273) | 1 hr | Slither per-class agreement |
| 23 | Verification: fp_estimator | `verification/fp_estimator.py` (336) | 1.5 hr | Stratified-by-(source, tier) FP rate |
| 24 | Verification: negative_checker | `verification/negative_checker.py` (235) | 1 hr | NonVulnerable Slither hit rate |
| 25 | Verification: gate + report | `verification/gate.py` (286) + `report_generator.py` (272) | 1.5 hr | Per-class verdict + 9-section markdown |
| 26 | Verification: probe_dataset | `verification/probe_dataset.py` (361) + `probe_trivials.py` (226) | 1.5 hr | 40+2 per class + 10 trivial positives |
| 27 | Verification: patterns | `verification/patterns/*.yaml` (10 files) | 30 min | Documentation only — semantic_checker doesn't read them |
| 28 | Splitting: splitters | `splitting/splitters.py` (441) | 1.5 hr | 4 splitter functions + `Contract`/`Splits` dataclasses |
| 29 | Splitting: dedup + leakage + cap | `splitting/dedup_enforcer.py` + `leakage_auditor.py` + `nonvulnerable_cap.py` | 1.5 hr | 3 safety nets |
| 30 | Registry: catalog | `registry/catalog.py` (541) | 1.5 hr | SQLite + YAML mirror; 4 base + 2 system tables |
| 31 | Registry: diff + lineage | `registry/dataset_diff.py` + `lineage_tracker.py` | 1 hr | Per-class metric projection + DAG |
| 32 | Analysis: feature_dist | `analysis/feature_dist.py` (436) | 1.5 hr | ⭐ The "Run 9 failure catcher" |
| 33 | Analysis: rest | `analysis/balance_viz.py` + `cooccurrence.py` + `overlap_detector.py` + `drift_monitor.py` | 1.5 hr | 4 read-only tools |
| 34 | Export: STUB | `export/__init__.py` + 6 stub files | 30 min | Awareness only — Stage 7 is not yet implemented |
| 35 | ADRs + design rationale | `docs/decisions/ADR-0001-*.md` + `ADR-0002-*.md` + `docs/v2-readiness-2026-06-12.md` | 1 hr | Why this package exists, what bugs existed at build start, the 7-gate check |
| 36 | Tests deep-dive | `tests/test_representation/test_byte_identical_regression.py` + `tests/test_verification/test_bccc_regression.py` + `ml/tests/test_sentinel_dataset.py` | 1.5 hr | The 5 critical tests — understand the merge gates |

**Estimated total:** ~35–40 hours. Adjust based on depth and audit density.

---

## 10. Cross-References

- **Global prefs:** `../../preferences.md` (P1–P16 + any P17+)
- **Global spec file protocol:** `../../reference.md` (the parent — at the project root)
- **Module's own prefs:** `./preferences.md`
- **Module's audit log:** `./audit_flags.md`
- **Module's session log:** `./session_log.md`
- **Source code root:** `~/projects/sentinel/data_module/`
- **Source README:** `~/projects/sentinel/data_module/README.md`
- **Source architecture:** `~/projects/sentinel/data_module/docs/architecture.md`
- **Source ADRs:** `~/projects/sentinel/data_module/docs/decisions/`
- **Source v2-readiness:** `~/projects/sentinel/data_module/docs/v2-readiness-2026-06-12.md`
- **Source config:** `~/projects/sentinel/data_module/config.yaml`
- **Source DVC DAG:** `~/projects/sentinel/data_module/dvc.yaml`
- **Source package:** `~/projects/sentinel/data_module/sentinel_data/`
- **Thin-adapter targets (must read alongside Stage 2):**
  - `~/projects/sentinel/ml/src/preprocessing/graph_schema.py`
  - `~/projects/sentinel/ml/src/preprocessing/graph_extractor.py`
  - `~/projects/sentinel/ml/src/data_extraction/windowed_tokenizer.py`

---

## 11. See also (other modules in this project)

This is `01_data_module_v2/`. When the data module is fully learned, the next
modules to set up:

- `02_preprocessing_ml/` (the ML preprocessing, not the data_module's preprocessing)
- `03_data_extraction_ml/`
- `04_datasets_ml/`
- `05_models_gnn/`
- `06_models_transformer/`
- `07_models_fusion_classifier/`
- `08_training/`
- `09_inference/`
- `10_contracts_solidity/`
