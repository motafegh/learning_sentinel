# Session 17 — Feature Engineering Deep Dive (`graph_extractor.py` signal functions)

**Date:** 2026-06-14
**Session number:** 17 of 36
**Mode:** Deep dive (400+ LOC of `_compute_*` / `_build_*` functions — signal design, normalization, sentinels)
**Estimated study time:** 45 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **12-dim feature vector** (Session 15): `[type_id, visibility, uses_block_globals, view, payable, complexity, loc, return_ignored, call_target_typed, has_loop, external_call_count, in_unchecked_block]`
- **Extraction flow** (Session 16): `extract_contract_graph` calls `_build_node_features` for declaration-level nodes and `_build_cfg_node_features` for CFG nodes. Both return 12-element lists.
- **Sentinel values**: Some features use -1.0 as "uncertain" (not assumed safe) to distinguish from confirmed 0.0 (safe) and 1.0 (dangerous).

---

## §1 Feature-by-Feature: the 12 Dimensions

### [0] `type_id` — Node Type (lines 1424, 930)

```python
float(cfg_type) / _MAX_TYPE_ID  # _MAX_TYPE_ID = 13.0 (v9)
```

All 14 node types (0–13) are normalized to [0, 1] by dividing by 13.0. `STATE_VAR(0)` → `0.0`. `CFG_NODE_ARITH(13)` → `1.0`.

### [1] `visibility` — Access Control (lines 1361–1363, 141–146)

```python
VISIBILITY_MAP = {
    "public": 0.0, "external": 0.0,
    "internal": 0.5, "private": 1.0,
}
```

**Ordinal encoding**: public/external are indistinguishable to the model (both mean "exposed"). internal is the midpoint. private is fully restricted. This lets the model learn "internal is halfway between exposed and fully hidden."

### [2] `uses_block_globals` — Block/Timestamp Signal (lines 620–676)

**Three-tier detection** (v9 Fix #2):

| Tier | Detection mechanism | Catches |
|---|---|---|
| 1 — Primary | `isinstance(rv, SolidityVariableComposed)` | `block.timestamp`, `block.number`, `block.difficulty`, `block.basefee`, `block.prevrandao` |
| 2 — Fallback A | Name-based (`rv.name in _BLOCK_NAMES`) | `now` (pre-0.8 alias for block.timestamp) |
| 3 — Fallback B | Type string contains "block" | Library-call wrappers (SafeMath-style) |

**Why three tiers?** Audit found 72.5% of BCCC Timestamp=1 graphs had `feat[2]=0` — the `now` keyword miss was the dominant cause (e.g., `roulette.sol`). Tier 1 matches `block.timestamp` (SolidityVariableComposed). Tier 2 matches the bare `now` keyword (parsed differently in pre-0.8 Solidity). Tier 3 matches wrappers where `block.timestamp` is assigned to a local variable.

**Per-CFG-node variant** (`_node_uses_block_globals`, lines 736–770): Identical logic but operates on a single Slither CFG node's IRs (not the entire function). Used by `_build_cfg_node_features` so each CFG node carries its own signal (a CALL node that happens to read timestamp gets flagged independently of other nodes in the function).

### [3] `view` — No State Modification (line 1386)

```python
1.0 if getattr(func, "view", False) else 0.0
```

A `view` function cannot modify state. Important for Reentrancy detection: a view function is unlikely to be the reentry point.

### [4] `payable` — Receives ETH (line 1387)

```python
1.0 if getattr(func, "payable", False) else 0.0
```

Payable functions can receive ETH. Combined with external calls, this is a prerequisite for Reentrancy. A non-payable function with external calls is still vulnerable (the call can trigger a receive fallback), so this feature adds signal weight, not determinism.

### [5] `complexity` — Cyclomatic Complexity (lines 1390–1394)

```python
_raw = float(len(func.nodes)) if func.nodes else 0.0
complexity = min(math.log1p(_raw) / math.log1p(100), 1.0)
```

**Log-normalized** from raw CFG block count (BUG-2 fix). Without normalization, a function with 100+ CFG blocks would have a raw complexity of 100 — dominating all other [0,1] features in dot products. `log1p(100) / log1p(100) = 1.0`. `log1p(10) / log1p(100) ≈ 0.68`.

**Contract-size normalization** (lines 1911–1915, E2):
```python
_size_factor = math.log1p(CONTRACT_SIZE) / CONTRACT_SIZE
x[:, 5] = x[:, 5] * _size_factor
```
After building all nodes, the complexity feature is scaled by `log1p(N)/N` where N is total graph nodes. This prevents large contracts from having inflated mean complexity relative to small contracts.

### [6] `loc` — Lines of Code (lines 1368–1372, 908–912)

```python
loc_raw = float(len(sm.lines)) if sm and sm.lines else 0.0
loc = min(math.log1p(loc_raw) / math.log1p(1000), 1.0)
```

Log-normalized with a higher saturation point (1000 lines vs 100 blocks for complexity). A typical function is 20–50 lines: `log1p(50)/log1p(1000) ≈ 0.57`.

### [7] `return_ignored` — Discarded Return Value (lines 261–336)

**Three possible values:**
- `1.0` — at least one external call's return value is never read after the call
- `0.0` — all external call return values ARE read after the call
- `-1.0` — SENTINEL: Slither IR unavailable for this function

**Why -1.0 instead of 0.0?** Closed-world assumption: "Slither couldn't analyze this function's IR" ≠ "return values are safe." The sentinel value lets the model distinguish "confirmed safe" from "unknown." The model learns to treat -1.0 as an intermediate risk signal.

**Detection logic (IMP-D1 fix, lines 314–334):** Scans IR operations in CFG topological order. For each external call (LowLevelCall, HighLevelCall, Send), checks if the lvalue variable is read by any later IR op in the same function:

```python
for call_idx, (_, op) in enumerate(all_ops_ordered):
    if not isinstance(op, (LowLevelCall, HighLevelCall, Send)):
        continue
    lval = op.lvalue
    if lval is None: return 1.0  # no lvalue → definitely ignored
    # Check if lval_name appears in any read AFTER this call
    used_after = any(
        getattr(rv, "name", None) == lval_name
        for _, later_op in all_ops_ordered[call_idx + 1:]
        for rv in (getattr(later_op, "read", None) or [])
    )
    if not used_after: return 1.0
return 0.0
```

**Per-CFG-node variant** (`_node_return_ignored`, lines 683–710): Scans only the IRs within a single Slither node, not the entire function. A CFG node with a single `call{value: x}()` that discards the return gets `return_ignored=1.0` even if the enclosing function has other nodes that DO check returns.

### [8] `call_target_typed` — Typed Interface vs Raw Address (lines 339–391)

- `1.0` — all external calls go through typed interfaces (`ContractType(receiver).call(...)`)
- `0.0` — any call goes to a raw address (`address(receiver).call{...}`)
- `-1.0` — SENTINEL: type resolution failed AND source mapping unavailable

**Why it matters:** Raw address calls are inherently more dangerous — they can target any contract, including malicious ones. Typed interface calls are restricted to known interface methods.

**Fallback source scan (lines 387–390):** When Slither's type resolution APIs fail, the function falls back to scanning the source text for `address(...).call{...}` patterns (excluding `address(this)` self-calls).

**Per-CFG-node variant:** `_node_call_target_typed` (lines 713–733) checks individual CFG node IRs. A CFG node with a `LowLevelCall` gets 0.0 even if other nodes in the function use typed calls.

### [9] `has_loop` — Loop Detection (lines 537–556)

```python
loop_types = {NodeType.IFLOOP, NodeType.STARTLOOP, NodeType.ENDLOOP}
for node in func.nodes:
    if node.type in loop_types: return 1.0
```

Detects `for`, `while`, `do-while` loops. Important for DenialOfService detection (loops that make external calls are DoS-prone). Falls back to Slither's `func.is_loop_present` attribute for older versions.

### [10] `external_call_count` — Outgoing Call Count (lines 589–617)

```python
n = len(high_level_calls) + len(low_level_calls)
# Plus Transfer/Send ops in each node's IRs
for op in node.irs:
    if isinstance(op, (Transfer, Send)): n += 1
return min(log1p(n) / log1p(20), 1.0)
```

Log-normalized with saturation at 20 calls. **Now includes Transfer and Send** (v9 Fix): because `payable(addr).transfer(amount)` inside a `distribute()` loop was classified as Transfer (not LowLevelCall), causing `external_call_count=0` for DoS-prone loops.

### [11] `in_unchecked_block` — Unchecked Scope (lines 408–450)

```
1.0 — at least one node in this function is inside an unchecked{} block (0.8+)
0.5 — pre-0.8 contract; unchecked keyword didn't exist (era proxy)
0.0 — all nodes are in a checked scope (0.8+, safe arithmetic)
```

**Three-tier encoding** (not binary like most features). The 0.5 sentinel is the model's way of distinguishing "confirmed unchecked" from "pre-0.8 where all arithmetic could overflow."

**Mechanism:** Slither 0.10's CFG parser sets `Scope(is_checked=False)` when entering an `UncheckedBlock` AST node. The feature checks `node.scope.is_checked` for each CFG node.

**Pre-0.8 contracts receive `in_unchecked=0.5`** for all function nodes (line 429):
```python
if is_pre_08:
    return 0.5
```

---

## §2 CFG Node Feature Inheritance (BUG-C3 fix, lines 887–897)

CFG nodes inherit function-scoped features from their parent FUNCTION node:

```
CFG node features:
  [0] type_id           — computed per-node (CFG_NODE_CALL, etc.)
  [1] visibility        ← inherited from parent FUNCTION
  [2] uses_block_globals — computed per-node (which statement reads block.*?)
  [3] view              ← inherited from parent FUNCTION
  [4] payable           ← inherited from parent FUNCTION
  [5] complexity        ← inherited from parent FUNCTION
  [6] loc               — computed per-node (line count of this statement)
  [7] return_ignored    — computed per-node (this call's return discarded?)
  [8] call_target_typed — computed per-node (typed interface or raw addr?)
  [9] has_loop          ← inherited from parent FUNCTION
  [10] external_call_count — computed per-node (calls in this statement)
  [11] in_unchecked     — computed per-node (this statement in unchecked?)
```

**Before the fix:** dims [1], [3], [4], [5], [9] were 0.0 for all CFG nodes. The model had no visibility/payable/complexity signal at the statement level. Since CEI violations are statement-level patterns (a CALL node followed by a WRITE node), the missing visibility signal made CEI detection near-impossible.

---

## §3 Normalization — Why Log1p?

Three features use `log1p` normalization:

| Feature | Saturation point | raw=1 | raw=10 | raw=100 |
|---|---|---|---|---|
| `complexity[5]` | log1p(100) ≈ 4.62 | 0.15 | 0.50 | 1.0 |
| `loc[6]` | log1p(1000) ≈ 6.91 | 0.10 | 0.33 | 0.67 |
| `external_call_count[10]` | log1p(20) ≈ 3.04 | 0.48 | 0.77 | 1.0 |

The saturation points differ by what's typical for each metric:
- Complexity rarely exceeds 100 CFG blocks per function
- LOC can reasonably hit 1000 (very long functions)
- External calls per function rarely exceed 20

---

## §0.6 Exercises (Q1–Q4)

### Q1 — `return_ignored` returns -1.0 as a sentinel. What other features use sentinel values and what do they mean?

`call_target_typed` also returns -1.0 (type resolution failure + source unavailable). `in_unchecked_block` uses 0.5 as a sentinel (pre-0.8 era proxy). All other features are binary (0.0/1.0) or continuous (complexity, loc, external_call_count in [0, 1]).

The sentinel pattern appears where the pipeline lacks information AND assuming safety (returning 0.0) would be dangerous. -1.0 preserves the uncertainty for the model to learn.

### Q2 — The `_compute_return_ignored` function scans all nodes in CFG topological order. What if the same lvalue is used by a later IR op that happens to be in a DIFFERENT function?

It can't — the scan is scoped to `all_ops_ordered` which is built from a single function's nodes. Cross-function return value analysis (callee returns a value, caller uses it) is not performed. This is an acknowledged gap: if function A calls B and B returns a value that A ignores, `return_ignored` for A's call to B is 1.0. But the check happens within A's IR — Slither inlines the call IR into the caller's function scope, so the lvalue IS checked against subsequent IR ops in A.

### Q3 — The `_detect_pre_08` function reads the first 2KB of the source file. What if the pragma appears after line 2000?

`pragma solidity` is almost always in the first few lines of any `.sol` file. If it appears after 2KB (extremely rare — that would require 2000 bytes of comment/NatSpec before the pragma), the detection falls back to `False` (assume 0.8+). This is the "safe default" — a pre-0.8 contract misdetected as 0.8+ would get `in_unchecked_block=0.0` instead of `0.5`, losing the era proxy signal but not producing false positives.

### Q4 — The contract-size normalization `_size_factor = log1p(N) / N` applies to ALL nodes' complexity features, including non-FUNCTION nodes (STATE_VAR, EVENT, etc.) that have `complexity=0`. Is this correct?

Yes — multiplying `0.0` by any factor is still `0.0`. The operation is a no-op for non-FUNCTION nodes. For FUNCTION nodes, the complexity is scaled down proportional to the contract's total node count. A medium-complexity function in a very large contract gets a lower complexity signal than the same function in a small contract. The rationale: in a large contract, any single function's complexity is less dominant relative to the total.
