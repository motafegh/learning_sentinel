# Recall R6 — Representation Remainder + Labeling (Sessions 19–22)

**Purpose:** Recall, review, relearn — consolidate 4 sessions into one
**Estimated review time:** 20 min
**Difficulty:** Medium

---

## 1. The Big Picture

Four modules complete the representation and labeling stages. The **CFG builder** produces a standalone JSON artifact (opt-in, for Stage 6 analysis). The **tokenizer** produces GraphCodeBERT windows `[W, 512]`. The **orchestrator** ties graph extraction + tokenization together with content-addressed caching. The **labeling system** reads source-specific label formats, merges across sources with tier precedence, and validates against minimum-viable-corpus thresholds.

---

## 2. Architecture Diagram

```
Representation (orchestrator.py):
  preprocessed/<source>/<sha256>.sol + .meta.json
    │
    └─ represent_source(name, cfg, data_dir)
         │
         ├── graph_extractor.extract_contract_graph()  → <sha256>.pt
         ├── tokenizer.tokenize_windowed_contract()     → <sha256>.tokens.pt
         └── Write <sha256>.rep.json (sidecar: versions + timing)
              │
              └── cache_manager.is_cached(sha256, schema_ver, extractor_ver)

Labeling (labeling/):
  per-source parsers:
    dive.py:         folder membership → canonical classes
    smartbugs_curated.py: hardcoded contract→label map
    solidifi.py:          CSV lookup + crosswalk

    ↓ per-contract .labels.json

  merger.py:
    ├── Tier precedence: T0 > T1 > T2 > T3 > T4
    ├── Within tier: positive > negative
    ├── DoS+Reentrancy co-occurrence noise check (>50% flags)
    └── Provenance tracking

    ↓ merged .labels.json

  gate.py:
    ├── total_contracts ≥ threshold
    ├── per_class positives ≥ thresholds (major: 200, minor: 50)
    └── CallToUnknown ≥ threshold (else: flag for review)
```

### Confidence Tiers

| Tier | Sources | Reliability |
|---|---|---|
| T0 | solidefi, defihacklabs | Manual audit — ground truth |
| T1 | smartbugs_curated, web3bugs | Curated dataset — verified |
| T2 | dive | Folder membership — likely correct |
| T3 | disl | Automated tool output |
| T4 | (future) | ML prediction — speculative |

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| Standalone CFG artifact (JSON, opt-in, per-function) | 19 | ☐ |
| GraphCodeBERT tokenizer (not CodeBERT) | 20 | ☐ |
| Sliding window: 512 tokens × 4 windows, stride 256 | 20 | ☐ |
| Content-addressed cache: (sha256, schema_ver, extractor_ver) | 21 | ☐ |
| `evict_stale()` — version mismatch detection | 21 | ☐ |
| DIVE parser: folder membership → canonical classes | 22 | ☐ |
| Crosswalk YAML: source folder → canonical class name | 22 | ☐ |
| Tier precedence — T0 > T1 > T2 > T3 > T4 | 22 | ☐ |
| DoS+Reentrancy co-occurrence noise (>50% flags) | 22 | ☐ |
| Provenance tracking in merged .labels.json | 22 | ☐ |
| Minimum viable corpus gate | 22 | ☐ |
| 3 criteria checked in gate (4th deferred to Stage 4) | 22 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: What's the difference between the CFG in graph_extractor (Session 18) and the standalone CFG builder (Session 19)?**

The graph_extractor's CFG is embedded in the PyG Data tensor — optimised for GNN message passing. The standalone CFG builder produces a JSON-serializable `CfgArtifact` with per-function subgraphs, string-typed node names, and loop back-edge counts. The primary consumer is Stage 6's `complexity_proxy_risk` detector, which needs a flat CFG view, not a GNN graph.

**Q2: The tokenizer uses a thin adapter pattern like graph_extractor. What tokenizer model does it use and why?**

GraphCodeBERT (`microsoft/graphcodebert-base`) — NOT CodeBERT. GraphCodeBERT is trained with data-flow awareness, making it better-suited for vulnerability detection. The sliding window approach (512 tokens, stride 256, up to 4 windows) ensures no code is clipped at boundaries.

**Q3: How does the labeling merger resolve a conflict where DIVE (T2) labels a contract "Reentrancy" and SolidiFi (T0) labels the same contract "IntegerUO"?**

Tier precedence: SolidiFi is T0, DIVE is T2. T0 > T2, so SolidiFi's label wins. The merged result is `{"classes": ["IntegerUO"], "tier": "T0", "provenance": [{"source": "solidifi", "tier": "T0", "classes": ["IntegerUO"]}, {"source": "dive", "tier": "T2", "classes": ["Reentrancy"]}]}`. The DIVE label is preserved in provenance for Stage 4 verification.

**Q4: The cache key is (sha256, schema_version, extractor_version). What invalidation scenarios does each component cover?**

- `sha256`: Content changes — new source file → new hash → cache miss. Same file → cache hit.
- `schema_version`: Schema changes — adding/removing a feature dimension. Bumping `FEATURE_SCHEMA_VERSION` invalidates all cached graphs.
- `extractor_version`: Code changes — bug fixes in feature computation that don't change the schema dimension. Bumping extractor version invalidates all cached graphs.

---

## 5. Session-by-Session Memory Triggers

- **S19:** `CfgNode`, `CfgEdge`, `CfgFunction`, `CfgArtifact`, DFS back-edge counting, opt-in via `--emit-cfg`
- **S20:** Tokenizer thin adapter, orchestrator flow, `<sha256>.rep.json` sidecar, deferred v3.1 features
- **S21:** `is_cached()`, `invalidate()`, `stale_entries()`, `evict_stale()`, three-way cache key
- **S22:** DIVE folder membership parser, crosswalk YAML, merger tier precedence, DoS+Reentrancy noise rule, gate criteria

---

## 6. What To Remember

1. **CFG builder is opt-in** — Slither adds ~0.5s/contract, disabled by default.
2. **GraphCodeBERT ≠ CodeBERT** — data-flow aware, sliding window with stride.
3. **Content-addressed cache** = (sha256, schema_ver, extractor_ver) — three independent axes.
4. **Label tier precedence** ensures highest-confidence source wins conflicts.
5. **Provenance tracking** preserves all source labels for verification audit trail.
6. **Gate** enforces minimum corpus quality before Stage 4 (verification).
