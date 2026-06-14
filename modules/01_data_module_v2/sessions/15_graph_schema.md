# Session 15 — Graph Schema (`representation/graph_schema.py`)

**Date:** 2026-06-14
**Session number:** 15 of 36
**Mode:** Understand (251 lines — single source of truth, locked class order, 12-dim feature schema)
**Estimated study time:** 35 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **Preprocessed .sol files** (Sessions 08–14): compiled, deduplicated, normalized Solidity contracts. These are the INPUT to graph extraction.
- **ContractMeta sidecar**: contains sha256, source_name, original_path, version_bucket — the context needed to link a graph back to its source
- **Graph-based GNN**: SENTINEL uses a graph neural network over program graphs. This file defines what goes INTO those graphs.

---

## §1 Why This File Exists — The Seam Swap (lines 12–24, 1–7)

Before Stage 7B (2026-06-12), every constant in this file was duplicated across:
- `ml/src/data_extraction/ast_extractor.py` — batch pipeline (~41K training graphs)
- `ml/src/inference/preprocess.py` — online inference (one contract per API request)

**The risk:** any divergence between the two copies would produce CORRECTLY-encoded training features but WRONGLY-encoded inference features — the model would silently produce wrong predictions with no error signal.

**The fix (Stage 7B seam swap):** Move all constants to `sentinel_data/representation/graph_schema.py`. Make `ml/src/preprocessing/graph_schema.py` a thin re-export shim. One edit propagates atomically to both pipelines.

**Interview answer:** "The seam swap pattern extracts a shared schema into its own module so training and inference can never drift apart. It's a structural guarantee enforced at import time."

---

## §2 Slither Version Assertion (lines 59–70)

```python
try:
    _ver_str = _importlib_metadata.version("slither-analyzer")
    _version = tuple(int(x) for x in _ver_str.split(".")[:3])
    if _version < (0, 9, 3):
        raise RuntimeError(
            f"slither-analyzer {_ver_str} is too old. "
            "SENTINEL requires >=0.9.3 for NodeType.STARTUNCHECKED support."
        )
except _importlib_metadata.PackageNotFoundError:
    pass  # slither not installed (inference-only deploy)
```

**Hard failure at import time**, not a log warning. If slither is installed and too old, the import crashes immediately. This is intentional — an old Slither silently produces wrong `in_unchecked_block` features by emitting a different node type enum set.

**The `PackageNotFoundError` catch** allows inference-only deploys (where slither isn't installed at all) to import the schema for its constants without requiring slither.

---

## §3 The 12-Dimensional Feature Vector (lines 83–84, 171–184)

```python
NODE_FEATURE_DIM: int = 12
FEATURE_NAMES: tuple[str, ...] = (
    "type_id",              # [0] node type as float (0.0–13.0, normalized to [0,1])
    "visibility",           # [1] public=0.0, external=0.0, internal=0.5, private=1.0
    "uses_block_globals",   # [2] 1.0 if references block.timestamp, block.number, msg.*, tx.*, now
    "view",                 # [3] 1.0 if function is view
    "payable",              # [4] 1.0 if function is payable
    "complexity",           # [5] cyclomatic complexity (McCabe)
    "loc",                  # [6] lines of code in the function body
    "return_ignored",       # [7] 1.0 if function has return value but caller ignores it
    "call_target_typed",    # [8] 1.0 if call target is a typed address (not msg.sender)
    "has_loop",             # [9] 1.0 if function contains for/while/do-while
    "external_call_count",  # [10] number of external calls originated from this node
    "in_unchecked_block",   # [11] 1.0 if this node is inside `unchecked { }` (v9 addition)
)
```

**Why 12 dimensions?** v1 had 8. Each version added features based on what the model was underperforming on:
- v2: added CFG node types (CFG_NODE_CALL/WRITE/READ/CHECK/OTHER)
- v9: added `in_unchecked_block[11]` and `CFG_NODE_ARITH(13)` — for IntegerUO detection
- v9: added `EXTERNAL_CALL` edge type — for Reentrancy detection

**Not all 12 features are used for all node types.** A `CONTRACT` node (type 7) has no `payable`, `has_loop`, or `external_call_count` — those features are zero-padded. The GNN learns to ignore irrelevant features per node type.

---

## §4 Node Types — 14 Categories (lines 91–135)

```python
NODE_TYPES = {
    # Declaration-level (ids 0–7, stable since v1)
    "STATE_VAR":   0,
    "FUNCTION":    1,
    "MODIFIER":    2,
    "EVENT":       3,
    "FALLBACK":    4,
    "RECEIVE":     5,
    "CONSTRUCTOR": 6,
    "CONTRACT":    7,
    # CFG subtypes (ids 8–12, added v2)
    "CFG_NODE_CALL":   8,
    "CFG_NODE_WRITE":  9,
    "CFG_NODE_READ":   10,
    "CFG_NODE_CHECK":  11,
    "CFG_NODE_OTHER":  12,
    # v9 addition for IntegerUO signal
    "CFG_NODE_ARITH":  13,
}
```

**Declaration-level (0–7):** These are the "declarations" — contract-level constructs. Functions, modifiers, events, etc. These form the COARSE graph structure.

**CFG subtypes (8–13):** These are FINE-GRAINED instruction-level nodes inside function bodies. The CFG builder (Session 17) splits each function body into basic blocks and classifies each block by its operation:
- `CALL`: external calls (e.g., `addr.transfer(x)`)
- `WRITE`: state variable writes (e.g., `balance[msg.sender] = x`)
- `READ`: state variable reads (e.g., `if (balance[msg.sender] > 0)`)
- `CHECK`: condition checks (e.g., `require(x > 0)`)
- `ARITH`: arithmetic operations (e.g., `a + b`, `balance[to] += amount`)
- `OTHER`: everything else

**`STRUCTURAL_PREFIX_TYPES`** (lines 129–135): The set of node types that ANY structural path must include when walking from contract root to a CFG node:

```python
STRUCTURAL_PREFIX_TYPES = frozenset({
    NodeType.FUNCTION, NodeType.MODIFIER, NodeType.CONSTRUCTOR,
    NodeType.FALLBACK, NodeType.RECEIVE,
})
```

---

## §5 Edge Types — 12 Relationships (lines 152–165)

```python
EDGE_TYPES: dict[str, int] = {
    "CALLS":             0,   # function A calls function B
    "READS":             1,   # function reads state variable
    "WRITES":            2,   # function writes state variable
    "EMITS":             3,   # function emits event
    "INHERITS":          4,   # contract inherits from contract
    "CONTAINS":          5,   # contract → function, function → CFG node
    "CONTROL_FLOW":      6,   # CFG edge (basic block → basic block)
    "REVERSE_CONTAINS":  7,   # runtime-only, NEVER on disk (computed at load time)
    "CALL_ENTRY":        8,   # CALLS → CONTAINS (cross-graph link)
    "RETURN_TO":         9,   # function exit → call site
    "DEF_USE":           10,  # variable def → variable use (data flow)
    "EXTERNAL_CALL":     11,  # v9 — call to external contract (untrusted target)
}
```

**`REVERSE_CONTAINS`** is a special edge: it's NEVER stored on disk. It's computed during graph loading by reversing all `CONTAINS` edges. This saves storage without losing navigability.

**`CALL_ENTRY`** (8) and **`RETURN_TO`** (9) form the inter-procedural call edges. `CALLS` says "function A calls function B." `CALL_ENTRY` connects the call site node (a CFG_NODE_CALL) to function B's entry node. `RETURN_TO` connects function B's exit back to the CFG node after the call site.

**`EXTERNAL_CALL`** (11, v9): distinguishes calls to untrusted external contracts from calls to known internal functions. Critical for Reentrancy detection.

---

## §6 Class Order — LOCKED (lines 190–209)

```python
CLASS_NAMES: list[str] = [
    "CallToUnknown",               # 0
    "DenialOfService",             # 1
    "ExternalBug",                 # 2
    "GasException",                # 3
    "IntegerUO",                   # 4
    "MishandledException",         # 5
    "Reentrancy",                  # 6
    "Timestamp",                   # 7
    "TransactionOrderDependence",  # 8
    "UnusedReturn",                # 9
]
NUM_CLASSES: int = 10
```

**Why locked:** The trained model's classifier head has 10 output neurons, one per class. The loss function computes cross-entropy between predicted logits[class_index] and the true label. If class order changes (e.g., Reentrancy moves from index 6 to index 3), every existing checkpoint predicts wrong labels without any error signal.

**The ADR-0009 transition:** Before Run 7, `NonVulnerable` was at index 9. It was removed because the model was trained on vulnerable-only contracts (the DIVE-positive class schema). The v9 checkpoint was trained with this 10-class order (no NonVulnerable). Changing CLASS_NAMES now requires bumping `FEATURE_SCHEMA_VERSION` and retraining.

---

## §7 Invariant Assertions (lines 222–234)

```python
assert len(FEATURE_NAMES) == NODE_FEATURE_DIM
assert len(EDGE_TYPES) == NUM_EDGE_TYPES
assert len(NODE_TYPES) == 14
assert max(NODE_TYPES.values()) == 13
```

These are **structural invariants** checked at import time. If someone adds a feature to `FEATURE_NAMES` without incrementing `NODE_FEATURE_DIM`, or adds a node type without updating the other constants, the import crashes.

**Why not tests?** These assertions run at import time, in every environment (dev, test, CI, production). A test-only check could be missed in a hotfix deploy. An import-time check cannot.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — The seam swap moved constants from `ml/src/preprocessing/graph_schema.py` to here. What happens if someone imports from the old path?

`ml/src/preprocessing/graph_schema.py` is now a thin re-export shim:
```python
from sentinel_data.representation.graph_schema import *  # noqa
```
All old imports continue to work. No code changes needed in any model file. This is a zero-disruption refactoring pattern.

### Q2 — `_MAX_TYPE_ID` is computed as `float(max(NODE_TYPES.values()))` = 13.0. Why is it a float?

Node type IDs are stored as the FIRST feature in the 12-dim vector (`type_id` at index 0). In the GNN, all features are normalized to [0, 1] range, so `type_id` is divided by `_MAX_TYPE_ID`:
```
normalized_type_id = node_type / _MAX_TYPE_ID
```
Thus `CFG_NODE_ARITH` (13) maps to `13.0 / 13.0 = 1.0`. `STATE_VAR` (0) maps to `0.0`. All 14 types are distributed in [0, 1].

### Q3 — `REVERSE_CONTAINS` is NEVER on disk. How is it computed?

The orchestrator or data loader reverses all `CONTAINS` edges at graph load time by iterating over the stored edge list and swapping source→target for edges of type CONTAINS (5). This produces `REVERSE_CONTAINS` (7) edges on the fly. The GNN needs reverse containment to walk from CFG nodes up to their containing functions, then up to the contract — without it, message passing only flows downward.

### Q4 — The Slither version check requires >=0.9.3. What breakage happens with 0.9.2?

Slither 0.9.2 doesn't emit `NodeType.STARTUNCHECKLED` (the node type marking the start of an `unchecked { }` block). Without this node type, the `in_unchecked_block[11]` feature is always 0 for all nodes — even nodes inside `unchecked { }`. The model gets zero signal for a feature it was trained on, producing degraded IntegerUO predictions. The hard version check prevents silent degradation.
