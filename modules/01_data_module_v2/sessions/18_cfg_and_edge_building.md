# Session 18 — CFG Building + Edge Construction (`graph_extractor.py` graph assembly)

**Date:** 2026-06-14
**Session number:** 18 of 36
**Mode:** Deep dive (CFG node typing, control flow edges, ICFG, DEF_USE, CEI detection)
**Estimated study time:** 40 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **14 node types** (Session 15): 8 declaration-level (0–7) + 6 CFG subtypes (8–13)
- **12 edge types** (Session 15): CALLS(0), READS(1), WRITES(2), EMITS(3), INHERITS(4), CONTAINS(5), CONTROL_FLOW(6), REVERSE_CONTAINS(7), CALL_ENTRY(8), RETURN_TO(9), DEF_USE(10), EXTERNAL_CALL(11)
- **Feature inheritance** (Session 17): CFG nodes inherit [1], [3], [4], [5], [9] from parent FUNCTION

---

## §1 `_cfg_node_type` — Mapping Slither Nodes to CFG Subtypes (lines 790–870)

Each Slither CFG node (a basic block) is classified into ONE of 6 CFG subtypes:

```
PRIORITY ORDER (highest to lowest):
  1. CFG_NODE_CALL  (8)  — external call (LowLevelCall, HighLevelCall, Transfer, Send)
  2. CFG_NODE_WRITE (9)  — state variable write
  3. CFG_NODE_READ  (10) — state variable read
  4. CFG_NODE_ARITH (13) — pure arithmetic (v9 Fix #4)
  5. CFG_NODE_CHECK (11) — require/assert/if condition
  6. CFG_NODE_OTHER (12) — everything else
```

**Why CALL > WRITE > READ > ARITH > CHECK?** The priority encodes vulnerability relevance. An external call is the SINGLE most important signal for Reentrancy/DoS detection — it must win even if the same node also writes state. A node with `balance[to] += amount` is BOTH arithmetic (the `+=` operation) AND a state write (balance is changed). WRITE wins because the state change is the dominant semantic for Reentrancy/CEI detection. ARITH only wins for purely arithmetic intermediate values like `x = a * b` (local variable, no state side effect).

**IR-based detection** (lines 825–856):
```python
# Priority 1: any IR op is an external call
if any(isinstance(op, (LowLevelCall, HighLevelCall, Transfer, Send)) for op in irs):
    return NODE_TYPES["CFG_NODE_CALL"]

# Priority 2: writes a state variable
sv_written = list(getattr(node, "state_variables_written", None) or [])
if sv_written or any(isinstance(op.lvalue, StateVariable) for op in irs):
    return NODE_TYPES["CFG_NODE_WRITE"]

# Priority 3: reads a state variable
sv_read = list(getattr(node, "state_variables_read", None) or [])
if sv_read or any(isinstance(v, StateVariable) for op in irs for v in op.read):
    return NODE_TYPES["CFG_NODE_READ"]

# Priority 4: pure arithmetic (no state side effect)
if _node_is_pure_arithmetic(slither_node):
    return NODE_TYPES["CFG_NODE_ARITH"]

# Priority 5: control-flow check node type
if node.type in {IF, IFLOOP, STARTLOOP, ENDLOOP, THROW}:
    return NODE_TYPES["CFG_NODE_CHECK"]
```

**Error handling (A10):** If any exception occurs during classification, the node falls through to `CFG_NODE_OTHER` and the `_cfg_type_fallback_count` is incremented. A high count in logs signals a Slither API version mismatch.

---

## §2 `_build_control_flow_edges` (lines 946–1047)

For each Slither Function object, this function:
1. Sorts CFG nodes by `(source_line, node_id)` — deterministic ordering across Slither versions
2. Assigns graph indices (Pass 1): `graph_idx = len(x_list)` (global index in the shared x_list)
3. Builds feature vectors via `_build_cfg_node_features`
4. Appends to `x_list` and `node_metadata`
5. Builds CONTAINS edges (function → CFG node): `contains_edges.append((func_node_idx, graph_idx))`
6. Builds CONTROL_FLOW edges (CFG node → successor) in Pass 2

**Index assignment is critical (lines 975–981):**
```python
graph_idx = len(x_list)  # CORRECT: next available global index
```
NOT `len(node_index_map)` — node_index_map only tracks CFG nodes for THIS function, not the full graph. Using it would produce overlapping indices across functions.

**Deterministic ordering (lines 989–997):**
```python
cfg_nodes = sorted(func.nodes, key=lambda n: (
    n.source_mapping.lines[0] if n.source_mapping and n.source_mapping.lines else 0,
    getattr(n, "node_id", 0),
))
```
Synthetic nodes (ENTRY_POINT, EXIT, etc.) have no source_mapping → get line 0 → sort first. This ensures identical source produces identical graphs across extraction runs and Slither versions.

**Dropped CONTROL_FLOW edges (A13, lines 1021–1045):** Cross-function sons (e.g., a node in function A has a successor in function B) are not in the local `node_index_map`. These edges are dropped and counted. The log helps detect if ICFG edge construction needs to handle these.

---

## §3 ICFG-Lite: Cross-Function Edges (`_add_icfg_edges`, lines 1050–1142)

Three edge types form the inter-procedural control flow graph:

### CALL_ENTRY (8): Calling CFG Node → Callee ENTRYPOINT (lines 1109–1113)
```python
for callee in sorted(node.internal_calls, key=lambda c: c.canonical_name):
    callee_entry = func_entry_map.get(callee_key)
    if callee_entry is not None:
        edges.append([caller_idx, callee_entry])
        edge_types.append(_CALL_ENTRY)
```

For every internal call (a function call within the same contract), a CALL_ENTRY edge connects the call site CFG node to the callee's ENTRYPOINT node.

### RETURN_TO (9): Callee Terminals → Call-Site Successors (lines 1115–1126)

After the callee finishes, control returns to the caller. Each terminal node (callee EXIT/ RETURN nodes with no successors) connects to each son of the call site node:
```python
for son in node.sons:  # successors of call site within the caller
    son_idx = local_map.get(son)
    for terminal_idx in callee_terminals:
        edges.append([terminal_idx, son_idx])
        edge_types.append(_RETURN_TO)
```

### EXTERNAL_CALL (11): Self-Loop on External Call Sites (lines 1128–1142, v9 Fix #3)

```python
high_lvl = list(getattr(node, "high_level_calls", None) or [])
low_lvl  = list(getattr(node, "low_level_calls", None) or [])
if high_lvl or low_lvl:
    edges.append([caller_idx, caller_idx])  # self-loop
    edge_types.append(_EXTERNAL_CALL)
```

External calls (cross-contract) have no corresponding function node in the graph (the callee is in a different contract/file). Instead of a real target, a self-loop on the call site CFG node marks it as external. The GNN's message passing treats self-loops as "this node has this property."

**Why self-loop instead of a virtual target node?** A virtual target would add extra nodes to the graph, shifting all subsequent indices. Self-loops preserve the graph structure while marking the node.

---

## §4 DEF_USE Data-Flow Edges (`_add_def_use_edges`, lines 1145–1265)

Tracks variable definitions and uses:

```
Def map (built in one pass over ALL functions):
  scope_key = (func_canonical_name, var_name)  — for local variables
  scope_key = (contract_canonical_name, var_name) — for state variables

For each function's CFG nodes:
  for each IR operation:
    for each read variable:
      if read variable matches a def scope_key → DEF_USE edge from def_node to use_node
```

**Two-tier scope key (A15 fix, lines 1157–1163):** Local variables use `(func, name)` scope — a variable `balance` in function A is DIFFERENT from `balance` in function B. State variables use `(contract, name)` scope — they ARE shared across functions.

**ReferenceVariable resolution (H-2 fix, lines 1176–1192):** Slither represents mapping/array lvalues (e.g., `balances[msg.sender]`) as `ReferenceVariable` — not a `StateVariable` or `LocalVariable`. The helper `_resolve_lval` follows `.points_to` → `.points_to_origin` to reach the underlying `StateVariable`.

---

## §5 CEI Path Detection (`_compute_has_cei_path`, lines 1268–1329)

Checks if any CFG_NODE_CALL can reach a CFG_NODE_WRITE within 8 CONTROL_FLOW hops:

```
Build adjacency list (CONTROL_FLOW edges only)
Find all CALL nodes and WRITE nodes
For each CALL node:
    BFS up to max_hops=8
    If WRITE node reached → return 1
return 0
```

**Why CONTROL_FLOW only (not CALL_ENTRY/RETURN_TO):** CEI is an intra-function pattern: "within this function, a call to an external contract happens before a state write." Following CALL_ENTRY into a callee would create cross-function false positives (caller calls, callee writes — but the write is in a different function's context, not a CEI violation).

**Why 8 hops?** In practice, a CEI violation has at most 3–5 CONTROL_FLOW hops between the call site and the state write. 8 covers 99.9% of true CEI patterns while bounding the BFS cost.

**Stored as `graph.has_cei_path` (int, 0 or 1).** Used by the auxiliary CEI loss after Gate 7.5 validates label quality.

---

## §6 Graph Index Layout Summary

```
x_list indices (fixed insertion order):
  ─
  0         CONTRACT node (selected contract)
  1..N      Parent CONTRACT nodes (inherited)
  N+1..M    STATE_VAR nodes
  M+1..P    MODIFIER nodes
  P+1..Q    EVENT nodes
  Q+1..R    FUNCTION 1 node
  R+1..S    CFG children of FUNCTION 1 (sorted by source_line)
  S+1..T    FUNCTION 2 node
  T+1..U    CFG children of FUNCTION 2
  ...
  ─
```

Edge types by source:
- **CALLS(0)**: declaration-level, built in step 6 of `extract_contract_graph`
- **READS(1)**: declaration-level, from `func.state_variables_read`
- **WRITES(2)**: declaration-level, from `func.state_variables_written`
- **EMITS(3)**: declaration-level, from `func.events_emitted` or IR fallback
- **INHERITS(4)**: declaration-level, from `contract.inheritance`
- **CONTAINS(5)**: CFG-level, from `_build_control_flow_edges`
- **CONTROL_FLOW(6)**: CFG-level, from `_build_control_flow_edges`
- **REVERSE_CONTAINS(7)**: runtime-only, computed during graph loading
- **CALL_ENTRY(8)**: ICFG-level, from `_add_icfg_edges`
- **RETURN_TO(9)**: ICFG-level, from `_add_icfg_edges`
- **DEF_USE(10)**: data-flow, from `_add_def_use_edges`
- **EXTERNAL_CALL(11)**: ICFG-level, from `_add_icfg_edges`

---

## §0.6 Exercises (Q1–Q4)

### Q1 — What happens if a Slither CFG node has BOTH an external call AND arithmetic IR ops?

The priority in `_cfg_node_type` is: CALL (1) > WRITE (2) > READ (3) > ARITH (4) > CHECK (5) > OTHER (6). A node with both CALL and ARITH ops is classified as `CFG_NODE_CALL` because the external call is the most vulnerability-relevant operation. The arithmetic signal for that node is lost at the TYPE level — but the `_build_cfg_node_features` function computes `external_call_count[10]` and `_node_in_unchecked[11]` independently of the node type, so the arithmetic-is-happening signal can still influence the model through other features.

### Q2 — `_add_icfg_edges` emits EXTERNAL_CALL self-loops. Could a self-loop on a CFG_NODE_CALL also match a WRITE node? (Both edges and self-loops go through the same edge_index tensor.)

No — self-loops use the same `caller_idx` for both source and target. A WRITE node has a different index. The edge_index tensor stores (source, target) pairs; a self-loop is (i, i). The GNN treats self-loops as identity edges (node message = transformed(node)). There's no ambiguity.

### Q3 — The DEF_USE builder uses `_resolve_lval` which follows `.points_to` → `.points_to_origin`. What if both are None?

If `points_to` is None and `points_to_origin` is None, the helper returns `None`, and the def is not recorded. The variable's definition is invisible to data-flow analysis. This typically happens for temporary variables or synthetic Slither IR variables that don't correspond to user-written code. The edge case was observed in practice with SolidiFI's complex mapping patterns.

### Q4 — The CEI BFS at line 1298 uses `cf_mask = (edge_attr == _CF)`. But what if the edge_attr tensor is empty (config.include_edge_attr=False)?

The code at lines 1294–1295 checks `if edge_index.shape[1] == 0: return 0`. An empty graph has zero edges, so CEI is trivially not present. But if `include_edge_attr` is False, `edge_attr` would be either empty or absent, and the mask `.cf_mask = (edge_attr == _CF)` would fail. This is a latent bug — it's only triggered if someone disables edge attributes, which doesn't happen in production. The fix would be to check `if not config.include_edge_attr: return 0` at the top of `extract_contract_graph` before calling `_compute_has_cei_path`.
