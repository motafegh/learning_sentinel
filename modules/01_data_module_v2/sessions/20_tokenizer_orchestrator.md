# Session 20 — Tokenizer + Orchestrator (`representation/tokenizer.py` + `orchestrator.py`)

**Date:** 2026-06-14
**Session number:** 20 of 36
**Mode:** Understand (72 + 344 lines — thin adapter + v2 orchestration runner)
**Estimated study time:** 25 min teaching + 10 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **Graph extraction** (Sessions 16–18): Converts .sol → PyG Data `.pt`
- **Thin adapter pattern** (Session 16): `sentinel_data/representation/graph_extractor.py` re-exports from `ml/`
- **Seam swap** (Session 15): Moving canonical code into `sentinel_data/representation/`

---

## §1 Tokenizer — Thin Adapter (72 lines)

```python
from ml.src.data_extraction.windowed_tokenizer import (
    tokenize_windowed_contract,
    init_worker,
    TOKENIZER_MODEL,    # "microsoft/graphcodebert-base"
    WINDOW_SIZE,        # 512
    STRIDE,             # 256
    MAX_WINDOWS,        # 4
)
```

Uses **GraphCodeBERT** (not CodeBERT). Key difference: GraphCodeBERT is trained with data-flow awareness, CodeBERT is not. The tokenizer produces `[max_windows, 512]` tensors — up to 4 windows of 512 tokens with stride 256.

**Why sliding window?** A single contract can exceed 512 tokens. Sliding windows with overlap (stride 256) ensure no code is clipped at window boundaries. The model processes all windows independently and aggregates via pooling.

---

## §2 Orchestrator — The Runner (344 lines)

### Input → Output

```
data/preprocessed/<source>/
    <sha256>.sol           →  (read by graph_extractor)
    <sha256>.meta.json     →  (read for sha256, source_name, version_bucket)

data/representations/<source>/
    <sha256>.pt            →  PyG Data graph
    <sha256>.tokens.pt     →  GraphCodeBERT token windows [W, 512]
    <sha256>.rep.json      →  Sidecar: schema_version, extractor_version, timing
```

### Flow

```
represent_source(name, cfg, data_dir):
  1. Resolve paths: preprocessed/<name>/ → representations/<name>/
  2. Load manifest or glob for *.meta.json files
  3. For each <sha256>.meta.json:
       a. Check cache: is_cached(sha256, schema_version, extractor_version)
       b. If cached: record hit, continue
       c. If not cached:
            - extract_contract_graph(sol_path, config) → Data
            - tokenize_windowed_contract(sol_path) → tokens
            - Write <sha256>.pt, <sha256>.tokens.pt, <sha256>.rep.json
  4. Return RepresentResult (contracts_seen, graphs_written, graphs_cached, ...)
```

### Key Design Decisions

**Content-addressed cache key**: `(sha256, schema_version, extractor_version)`. Three-way key means:
- Changing the schema (adding a feature) → cache miss → re-extract all graphs
- Changing the extractor (fixing a bug) → cache miss → re-extract all graphs
- Same sha256 = same contract content → cache hit

**Sidecar `.rep.json`** structure:
```json
{
    "sha256": "...",
    "source_name": "dive",
    "schema_version": "v9",
    "extractor_version": "2.0.0",
    "extraction_duration_s": 1.234,
    "n_nodes": 42,
    "n_edges": 87,
    "has_cei_path": 1
}
```

**What's NOT in the orchestrator** (deferred to Task 2.8, Day 4):
- Cache invalidation logic (handled by `cache_manager.py`)
- CFG / PDG / call-graph / opcode builders (v3.1 for most)
- Multiprocessing (serial for now; will use `mp.Pool` similar to `preprocessing/parallel.py`)
