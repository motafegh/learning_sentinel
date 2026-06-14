# Recall R5 — Graph Extraction (Sessions 15–18)

**Purpose:** Recall, review, relearn — consolidate 4 sessions into one
**Estimated review time:** 25 min
**Difficulty:** Core (architecture + feature engineering — must know for interviews)

---

## 1. The Big Picture

Graph extraction converts preprocessed `.sol` files into PyG `Data` graph objects. The **schema** (`graph_schema.py`) is the single source of truth: 14 node types, 12 edge types, 12-dim feature vectors, 10 locked class names. The **extractor** (`graph_extractor.py`, 2056 LOC in `ml/`) runs Slither on each `.sol` file and builds a program graph with declaration-level nodes (CONTRACT, FUNCTION, STATE_VAR) and CFG-level nodes (CALL, WRITE, READ, ARITH, CHECK, OTHER). Feature vectors encode vulnerability-relevant signals. Edges encode control flow, data flow, containment, and call relationships.

---

## 2. Architecture Diagram

```
graph_schema.py (single source of truth)
  │
  ├── NODE_TYPES: 14 types (0-13)
  ├── EDGE_TYPES: 12 types (0-11)
  ├── FEATURE_NAMES: 12-dim vector
  ├── CLASS_NAMES: 10 locked classes
  └── Slither version assertion (>=0.9.3)

graph_extractor.py (thin adapter → ml/src/preprocessing/graph_extractor.py)
  │
  └─ extract_contract_graph(sol_path, config) → Data
       │
       ├── 1. Slither instantiation (detectors_to_run=[])
       │
       ├── 2. _select_contract(sl, config)
       │       most_derived (92%) > last (87.4%) > most_funcs (47.4%)
       │
       ├── 3. Pre-0.8 detection
       │
       ├── 4. Build declaration nodes (fixed order):
       │       CONTRACT → parents → STATE_VARs → MODIFIERs → EVENTs → FUNCTIONs
       │
       ├── 5. For each FUNCTION:
       │     └─ _build_control_flow_edges()
       │          ├── CFG nodes appended to x_list
       │          ├── CONTAINS edges (function → CFG node)
       │          └── CONTROL_FLOW edges (node → successor)
       │
       ├── 6. Cross-function edges:
       │     ├── _add_icfg_edges()  → CALL_ENTRY, RETURN_TO, EXTERNAL_CALL
       │     └── _add_def_use_edges() → DEF_USE data-flow edges
       │
       ├── 7. Declaration-level edges:
       │       CALLS, READS, WRITES, EMITS, INHERITS
       │
       ├── 8. Validate + assemble PyG Data
       │     └── CEI path detection (BFS)
       │
       └── Return Data(x, edge_index, edge_attr, node_metadata, ...)
```

### Feature Vector (12 dims)

| idx | Name | Source | Sentinel |
|---|---|---|---|
| 0 | `type_id` | Node type / 13.0 | — |
| 1 | `visibility` | public=0, external=0, internal=0.5, private=1.0 | — |
| 2 | `uses_block_globals` | Three-tier detection | — |
| 3 | `view` | Slither Function.view | — |
| 4 | `payable` | Slither Function.payable | — |
| 5 | `complexity` | log1p(CFG blocks) / log1p(100) | — |
| 6 | `loc` | log1p(lines) / log1p(1000) | — |
| 7 | `return_ignored` | LValue read after call? | **-1.0**: unknown |
| 8 | `call_target_typed` | Typed interface vs raw address | **-1.0**: unknown |
| 9 | `has_loop` | Loop construct present | — |
| 10 | `external_call_count` | log1p(calls) / log1p(20) | — |
| 11 | `in_unchecked_block` | node.scope.is_checked | **0.5**: pre-0.8 era |

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| Schema is single source of truth (seam swap) | 15 | ☐ |
| Locked CLASS_NAMES — reordering invalidates checkpoints | 15 | ☐ |
| Slither version assertion (hard failure at import) | 15 | ☐ |
| 14 node types: 8 declaration + 6 CFG | 15 | ☐ |
| 12 edge types + REVERSE_CONTAINS (runtime-only) | 15 | ☐ |
| `most_derived` heuristic (92% vs 47.4% for most_funcs) | 16 | ☐ |
| Synthetic keys for overriding functions (NF-10) | 16 | ☐ |
| Fixed node insertion order (indices must be stable) | 16 | ☐ |
| Three-tier `uses_block_globals` detection | 17 | ☐ |
| Log-normalization (loc, complexity, ext_call_count) | 17 | ☐ |
| CFG node feature inheritance from parent FUNCTION (BUG-C3) | 17 | ☐ |
| CFG node type priority: CALL > WRITE > READ > ARITH > CHECK > OTHER | 18 | ☐ |
| CEI path detection: BFS over CONTROL_FLOW edges only | 18 | ☐ |
| DEF_USE two-tier scope key (local vs state) | 18 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: What is the "seam swap" and why was it needed?**

Before Stage 7B, all graph constants (NODE_TYPES, EDGE_TYPES, FEATURE_NAMES, CLASS_NAMES) were DUPLICATED across `ml/src/data_extraction/ast_extractor.py` (training) and `ml/src/inference/preprocess.py` (inference). A change to one required a manual sync to the other — divergence would silently corrupt inference. The seam swap moved all constants to `sentinel_data/representation/graph_schema.py` and made the `ml/` path a thin re-export shim. Now one edit propagates to both pipelines atomically.

**Q2: The feature vector has 12 dimensions. Why 12 and not 8 or 16?**

Each v1-v9 version added features based on what the model was underperforming on:
- v2: Added CFG node types (5 new) + CONTAINS/CONTROL_FLOW edges
- v7: Dropped `in_unchecked` (dead signal for 87.9% of contracts)
- v9: Re-added `in_unchecked` (proper Slither 0.10 detection) + `CFG_NODE_ARITH` + `EXTERNAL_CALL` edge
- Each new feature was an AUDIT-driven response to a specific false-positive pattern

Not all 12 features are used for all node types — CONTRACT nodes have zero-padded `payable`, `has_loop`, etc. The GNN learns to ignore irrelevant features per node type.

**Q3: What's the CFG node type priority and why CALL > WRITE > READ?**

Priority encodes vulnerability relevance. A node with `balance[to] += amount` is BOTH an arithmetic operation AND a state write. WRITE wins because the state change is the dominant semantic for Reentrancy/CEI detection. CALL wins over everything because an external call is the single most important signal — even if the same node also writes state, the model MUST see the CALL type.

**Q4: Why does `return_ignored` return -1.0 as a sentinel instead of 0.0?**

Closed-world assumption: "Slither couldn't analyze this function's IR" ≠ "return values are safe." Returning 0.0 would mean "confirmed safe" when it actually means "unknown." The -1.0 sentinel preserves the uncertainty. The model learns to treat -1.0 as an intermediate risk. The same pattern applies to `call_target_typed`.

---

## 5. Session-by-Session Memory Triggers

- **S15:** `graph_schema.py` constants, seam swap, Slither >=0.9.3, locked class order, invariant assertions
- **S16:** `extract_contract_graph` flow, typed exceptions, `_select_contract`, fixed insertion order, parent writes meta
- **S17:** 12 features in detail, log-normalization, sentinel values, CFG feature inheritance, three-tier block globals
- **S18:** CFG node type priority, ICFG-Lite (CALL_ENTRY, RETURN_TO, EXTERNAL_CALL), DEF_USE data-flow, CEI BFS

---

## 6. What To Remember

1. **Schema is locked** — changing CLASS_NAMES order invalidates checkpoints.
2. **Seam swap** ensures training/inference can never drift apart.
3. **12-dim feature vector** with log-normalization for scale-invariant learning.
4. **-1.0 sentinels** preserve uncertainty (not assumed safe).
5. **CFG node type priority** encodes vulnerability relevance.
6. **ICFG-Lite** (CALL_ENTRY + RETURN_TO) enables cross-function reasoning.
7. **CEI path** is intra-function only — no cross-function false positives.
