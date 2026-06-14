# Session 16 — Graph Extractor (`representation/graph_extractor.py` + `ml/` backend)

**Date:** 2026-06-14
**Session number:** 16 of 36
**Mode:** Deep dive (2056 lines — Slither-to-PyG conversion, feature extraction, edge building)
**Estimated study time:** 50 min teaching + 20 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **Graph Schema** (Session 15): 14 node types, 12 edge types, 12-dim feature vector, locked 10-class vocabulary. This file produces PyG `Data` objects matching that schema.
- **Thin adapter pattern** (Session 16 intro): `sentinel_data/representation/graph_extractor.py` re-exports from `ml/src/preprocessing/graph_extractor.py`. The real code lives in `ml/`.
- **Preprocessed .sol files** (Sessions 08–14): the input. Every `.sol` file has a sibling `.meta.json` with compiler version, sha256, etc.

---

## §1 Architecture — Two Files, One Implementation

```
sentinel_data/representation/graph_extractor.py  (77 lines — thin adapter)
    │
    └── re-exports from: ml/src/preprocessing/graph_extractor.py  (2056 lines)
        │
        ├── extract_contract_graph(sol_path, config) → Data     (line 1567)
        ├── GraphExtractionConfig                                 (line 211)
        ├── GraphExtractionError hierarchy                        (line 163)
        └── _compute_* / _build_* feature helpers                 (lines 261–1559)
```

**Why the adapter?** (from the docstring):
> - Zero code duplication. Bug fixes in `ml/` propagate to the new path automatically.
> - The Stage 7 seam swap is a 1-line change (delete this file) instead of a multi-file refactor.
> - The byte-identical-output guarantee is trivially true — same object, different import name.

**`__getattr__` fallback** (lines 47–63): If `ml/` is not on the Python path, the adapter lazily imports it and raises a descriptive error. This lets `sentinel-data` become a standalone PyPI package in the future (infers without requiring `ml/`).

---

## §2 Exception Hierarchy (lines 163–204)

```python
GraphExtractionError          — base class
 ├── SolcCompilationError     — user error → HTTP 400 (bad Solidity)
 ├── SlitherParseError        — infra error → HTTP 500 (Slither bug)
 └── EmptyGraphError          — user error → HTTP 400 (no AST nodes)
```

**Never returns None. Always raises a typed exception.**

- **`SolcCompilationError`**: Solidity syntax error, wrong pragma, missing import. The file exists but solc rejects it. The inference API translates this to "you sent bad Solidity" (HTTP 400). The batch pipeline logs and skips.
- **`SlitherParseError`**: Slither itself crashes. Root causes: Slither API version mismatch, exotic contract pattern, OS-level failure. The inference API translates this to "internal error" (HTTP 500). The batch pipeline logs and skips.
- **`EmptyGraphError`**: File parsed successfully but produced zero non-dependency AST nodes. Typically: the file only contains `import` statements with no contract body. GNNEncoder can't process a zero-node graph.

**Exception routing (lines 1624–1646):** The entry point catches the raw Slither exception, checks if it's a `SlitherError` (via `isinstance`) or a `CompilationError` (via string matching on error text), and re-raises as the appropriate typed exception.

---

## §3 `GraphExtractionConfig` (lines 211–254)

```python
@dataclass
class GraphExtractionConfig:
    multi_contract_policy: str = "most_derived"
    target_contract_name: str | None = None
    include_edge_attr: bool = True
    solc_binary: str | Path | None = None
    solc_version: str | None = None
    allow_paths: str | None = None
```

**`multi_contract_policy`** — controls which contract to analyze when a `.sol` file defines multiple:

| Policy | Method | BCCC Accuracy | When to use |
|---|---|---|---|
| `"most_derived"` | Picks the contract that inherits from the most in-file contracts | ~92% | **Default.** The vulnerable contract inherits library contracts defined earlier |
| `"last"` | Picks the last non-interface contract | 87.4% | Simple fallback when inheritance info is absent |
| `"most_funcs"` | Picks the contract with the most functions | 47.4% | **LEGACY — worse than random!** Kept for compatibility |
| `"by_name"` | Uses `target_contract_name` to select | N/A | For targeted extraction (inference API with known contract name) |

**Why "most_derived" beats "most_funcs":** In BCCC, vulnerable contracts follow the pattern:
```solidity
contract StandardToken { ... }
contract ERC20Token is StandardToken, Ownable { ... }  // ← vulnerable one
```
`ERC20Token` has FEWER functions (inherits most from `StandardToken`), so "most_funcs" picks `StandardToken`. But `ERC20Token` inherits from `StandardToken`, so "most_derived" picks `ERC20Token`. 92% vs 47.4%.

**`_build_solc_args` (lines 1546–1560):** Converts config into `--allow-paths .,<path>` for Slither. Only applies to solc >=0.5 (older solc doesn't support `--allow-paths`).

---

## §4 `extract_contract_graph()` — The Main Flow (lines 1567–2056)

### Step 1: Slither Availability + Instantiation (lines 1608–1646)

```python
try:
    from slither import Slither
except ImportError as exc:
    raise RuntimeError("Slither is not installed. Run: pip install slither-analyzer")
```

Slither is imported LOCALLY (not at module level). Why? So `sentinel_data` can be imported without requiring Slither in the environment. Only the graph extraction pipeline imports it.

**`detectors_to_run=[]`** — we suppress all Slither detectors. We only use Slither for its AST parser and IR generator. Running detectors would waste CPU time.

### Step 2: Contract Selection (line 1649, `_select_contract` at line 1444)

```python
contract = _select_contract(sl, config)
```

Filters out dependency-only contracts (`is_from_dependency()`), then applies the `multi_contract_policy` heuristic. For `"most_derived"`:
1. Filter non-interface contracts from non-dependency candidates
2. Score each by how many other candidates it inherits from (tiebreak: later source order wins)
3. Pick the highest-scored contract

### Step 3: Pre-0.8 Detection (line 1657, `_detect_pre_08` at line 559)

```python
_is_pre_08: bool = _detect_pre_08(config.solc_version, sol_path)
```

Checks config version first, then falls back to scanning the first 2KB of the source for `pragma solidity`. This affects the `in_unchecked_block` feature encoding: pre-0.8 contracts get 0.5 (era proxy) instead of 0.0 or 1.0.

### Step 4: Build Shared Node Lists (lines 1660–1877)

```
x_list:        list = []  # feature vectors [N, 12]
node_metadata: list = []  # index-aligned dicts
node_map:      dict = {}  # canonical_name → index (declaration nodes only)
```

**Fixed insertion order (lines 1699–1713):**
1. CONTRACT node (the selected contract)
2. Parent CONTRACT nodes (inherited contracts) — so INHERITS edges resolve (BUG-H8)
3. STATE_VAR nodes
4. MODIFIER nodes (registered BEFORE edge creation — BUG-H9)
5. EVENT nodes (registered BEFORE edge creation — BUG-H9)
6. For each FUNCTION: function node → immediately its CFG children (sorted by source_line)

This order is the **graph node index layout**. Changing it changes the edge_index and invalidates existing checkpoints.

**`_add_node` helper (line 1670):**
```python
def _add_node(obj, initial_type_id, override_key=None):
    key = override_key or obj.canonical_name or obj.name
    if key in node_map: return None  # dedup by canonical_name
    idx = len(x_list)
    node_map[key] = idx
    x_list.append(_build_node_features(...))
    node_metadata.append({"name": ..., "type": ..., "source_lines": ...})
    return idx
```

**Synthetic keys for overriding functions (NF-10, lines 1748–1768):** If two functions have the same `canonical_name` (overriding/overloaded), the second gets a synthetic key `f"{name}__override__{func_index}"` so both are preserved. Crucial for vulnerability detection — overrides often introduce bugs.

### Step 5: Extract CFG + Edges (lines 1717–1847)

For each function:
1. `_build_control_flow_edges()` → builds CFG_NODE children, CONTAINS edges (function→CFG node), CONTROL_FLOW edges (CFG node→CFG node successor)
2. Record `_func_entry_map`, `_func_terminal_map`, `_func_cfg_maps` for ICFG edge construction
3. After all functions processed:
   - `_add_icfg_edges()` → CALL_ENTRY, RETURN_TO, EXTERNAL_CALL edges
   - `_add_def_use_edges()` → DEF_USE data-flow edges

### Step 6: Declaration-Level Edges (lines 1917–2007)

Iterates over contract functions and emits:
- **CALLS**: function → internal call target
- **READS**: function → state variable read
- **WRITES**: function → state variable written
- **EMITS**: function → event emitted (with fallback for pre-0.4.21 Solidity — BUG-H7)
- **INHERITS**: contract → parent contract

### Step 7: Feature Tensor + Validation (lines 1869–1916)

```python
x = torch.tensor(x_list, dtype=torch.float)  # [N, 12]
assert x.shape[1] == NODE_FEATURE_DIM  # 12
```

**Out-of-range (OOR) detection (BUG-L4, lines 1879–1892):** Any feature outside [-1, 1] is logged as a warning. A single bad contract doesn't crash the run, but OOR features destabilize training. The log helps identify which contracts produce bad features.

**CEI path computation (line 2049, `_compute_has_cei_path` at line 1268):** BFS from every CFG_NODE_CALL to any CFG_NODE_WRITE within 8 CONTROL_FLOW hops. Used by the auxiliary CEI loss in Phase 7.

### Step 8: Assemble PyG Data (lines 2034–2056)

```python
graph = Data(x=x, edge_index=edge_index)
graph.edge_attr = edge_attr  # if include_edge_attr
graph.has_cei_path = cei_val # 0 or 1
graph.node_metadata = node_metadata
graph.contract_name = contract.name
graph.num_nodes = x.shape[0]
graph.num_edges = len(edges)
return graph
```

---

## §5 Telemetry Counters (lines 133–138, 1818–1825)

Module-level counters accumulate failures across all files in a process:

```python
_call_target_fail_count: int = 0     # A6: type-resolution failures
_cfg_type_fallback_count: int = 0    # A10: Slither-import failures
_ext_call_fail_count: int = 0        # NF-7: external_call_count failures
_block_globals_fail_count: int = 0   # NF-7: block_globals failures
_in_unchecked_fail_count: int = 0    # NF-8: in_unchecked failures
```

These are logged at batch end by the orchestrator. A high count for any counter signals Slither API version drift — the feature is silently defaulting to 0.0 for many files.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — The `_add_node` dedup key is `canonical_name`. What happens when two contracts or functions have the same canonical_name but different bodies?

For contracts: `_add_node` returns `None` for the duplicate → the second contract's node is lost. This matters for inherited contracts (BUG-H8 fix at lines 1705–1706 adds parent CONTRACT nodes — if two parents have the same canonical_name, the second won't be added, and its INHERITS edge can't resolve).

For functions: The NF-10 fix (line 1753) assigns a synthetic key `f"{name}__override__{func_index}"` to the second occurrence, ensuring both functions appear as separate nodes. This was added because override functions in SolidityFi patterns would silently merge, losing the vulnerability signal in the overriding function.

### Q2 — `detectors_to_run=[]` is passed to Slither. What detectors would be useful if we enabled them?

Slither has ~90+ detectors (reentrancy, uninitialized state, tx.origin, etc.). If we enabled them, the graph could include detector outputs as additional features (e.g., "Slither flagged this as reentrancy-risk"). However, this creates a circular dependency: the model would be trained on features produced by the very detectors it's trying to learn. Current design keeps detectors disabled for purity — the GNN learns patterns from raw AST structure, not from heuristic labels.

### Q3 — The `_is_pre_08` flag affects `in_unchecked_block` encoding: pre-0.8 gets 0.5, 0.8+ gets 0.0 or 1.0. Why 0.5 instead of 0.0?

Slither marks ALL CFG nodes in pre-0.8 contracts as `is_checked=False` (an artefact of the parser — the `unchecked` keyword didn't exist). If the model received `in_unchecked=1.0` for pre-0.8 contracts, it would learn "pre-0.8 = unchecked = overflow risk" — which is true but conflates "confirmed unchecked block" with "pre-0.8 era where all arithmetic could overflow." The 0.5 sentinel preserves the distinction: "arithmetic risk unknown due to era."

### Q4 — The CEI path BFS (line 1268) traverses CONTROL_FLOW edges only (not CALL_ENTRY/RETURN_TO). Why not follow cross-function calls?

A CEI violation is defined as: "an external call to an untrusted contract happens BEFORE a state write within the same function." Following CALL_ENTRY into the callee would find CEI violations across function boundaries (caller does call, callee does write) — but this is NOT a CEI pattern because the callee's write happens in its own execution context. The caller can still safely apply CEI within its own function. The intra-function restriction is conservative — it finds true CEI violations without cross-function false positives.
