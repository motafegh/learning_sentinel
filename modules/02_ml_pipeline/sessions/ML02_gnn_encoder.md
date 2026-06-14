# ML02 — GNN Encoder: Line-by-Line Three-Phase GAT Architecture

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML02 of 10 (REWRITE — deep source-code-chunk edition)
**Source:** `ml/src/models/gnn_encoder.py` (667 lines)
**Status:** 🟢 Complete

---

## How to read this doc

This document follows the source file **line by line, chunk by chunk**. Each chunk maps to a contiguous block of the file. Read with `gnn_encoder.py` open side by side.

---

## CHUNK 1: Module docstring + global constants (lines 1–62)

### Lines 1–43 — Architecture docstring

The docstring is the **single source of truth** for the three-phase design. It's not a comment — it's a reference that every person joining the project reads first.

```
Phase 1 (Layers 1+2): Structural aggregation + input skip (IMP-G2)
  Edges: types 0–5 (CALLS, READS, WRITES, EMITS, INHERITS, CONTAINS)
  add_self_loops=True
  BUG-R7-2: type_embedding nn.Embedding(13,16) prepended → _GNN_IN_DIM=27 input.
  Layer 1: _GNN_IN_DIM→hidden_dim (concat 8 heads) + input_proj skip (IMP-G2)
  Layer 2: hidden_dim→hidden_dim (concat 8 heads) + residual
  IMP-G2: input_proj skip = Linear(27, 256, bias=False). Prevents raw feature loss.

Phase 2 (Layers 3+4+5): Layer-specific CFG + ICFG (IMP-G1, IMP-R7-1)
  add_self_loops=False  ← CRITICAL — self-loops cancel directional signal
  IMP-R7-1: heads=4, concat=True, out=64/head → output stays hidden_dim (256).
  IMP-G1: each layer processes a DISTINCT edge subset.
  Layer 3 (conv3):  CONTROL_FLOW(6) only — intra-function execution ordering
  Layer 4 (conv3b): CALL_ENTRY(8) + RETURN_TO(9) only — cross-function call structure
  Layer 5 (conv3c): CF(6)+CALL_ENTRY(8)+RETURN_TO(9) joint — integration layer

Phase 3 (Layers 6+7+8): Bidirectional CONTAINS (IMP-G3)
  heads=1, concat=False. Upward and downward CONTAINS passes.
  Layer 6 (conv4):  REVERSE_CONTAINS up — CFG→FUNCTION (Phase 2 signal rises)
  Layer 7 (conv4b): REVERSE_CONTAINS up — second hop (multi-function patterns)
  Layer 8 (conv4c): CONTAINS down — FUNCTION→CFG (IMP-G3: distributes enriched context)

JK Connections (Phase 1-A1): learned attention aggregation over all three phase outputs.
Per-Phase LayerNorm (Phase 1-A2): prevents Phase 1 norm from dominating JK softmax.
```

**Key insight — why 2+3+3?** The layer counts correspond to graph-theoretic distances:

| Phase | Layers | Max propagation distance | What it captures |
|---|---|---|---|
| 1 (structural) | 2 | 2 hops | Within-function neighborhood of each node |
| 2 (CFG) | 3 | 3+3+3 = 9 CF steps | ENTRY→CALL→TMP→WRITE CEI sequence (4-5 steps) |
| 3 (CONTAINS) | 3 | 2 up + 1 down = tree height | CFG→FUNCTION→CFG cross-function patterns |

The docstring also gives the parameter listing (lines 31-42):

```python
in_channels   = _GNN_IN_DIM (28 = NODE_FEATURE_DIM 12 + type_emb 16; model-internal v9)
hidden_dim    = 256
heads         = 8 (Phase 1); 4 (Phase 2, IMP-R7-1); 1 (Phase 3)
dropout       = 0.2
use_edge_attr = True
edge_emb_dim  = 64
num_layers    = 8 (2+3+3 phases)
use_jk        = True / jk_mode = 'attention'
```

**Total trainable params: ~2.5M** (for the GNN alone — CodeBERT is 125M frozen).

### Lines 45–52 — Imports

```python
from ml.src.preprocessing.graph_schema import NODE_FEATURE_DIM, NODE_TYPES, NUM_EDGE_TYPES, EDGE_TYPES
```

Only `GATConv` is used from PyG — not `GCNConv`, not `SAGEConv`. GAT's attention mechanism is critical because different nodes (CALL nodes vs WRITE nodes) need different importance weights for their neighbors.

### Lines 53–59 — Input dimension constants

```python
_TYPE_EMB_DIM: int = 16
_NUM_NODE_TYPES: int = int(max(NODE_TYPES.values())) + 1  # 14 for v9 schema (IDs 0–13)
_GNN_IN_DIM:   int = NODE_FEATURE_DIM + _TYPE_EMB_DIM    # 12 + 16 = 28 (v9)
```

**What the comments DON'T tell you (source-code truth):**

The comment on line 57 says `_NUM_NODE_TYPES: int = int(max(NODE_TYPES.values())) + 1` — this is NOT hardcoded to 14. It computes at import time by taking the max of the `NODE_TYPES` dict. Since `NODE_TYPES` includes `CFG_NODE_ARITH: 13`, `max(...)` returns 13, plus 1 = 14. If a new node type is added to `NODE_TYPES`, this auto-adjusts.

**Critical bug that was fixed (BUG-R7-2, line 54-55 comment):** Originally node type was stored as `float(type_id) / 12.0` in feat[0]. GATConv sees this as a continuous scalar — it cannot learn 14 distinct categorical representations from a single float. The `type_embedding` fixes this by mapping the recovered integer type ID to a 16-dim learned vector, giving each node type its own 16-dim representation.

### Lines 61–62 — Architecture lock

```python
SENTINEL_GNN_NUM_LAYERS: int = 8
```

[A27] This is enforced in `GNNEncoder.__init__` with a `ValueError` if `num_layers != 8`. The architecture is FIXED at 8 layers because the three-phase design is structurally tied to the smart contract graph's hierarchy. You cannot just add a layer — it would need a new phase.

---

## CHUNK 2: `_JKAttention` — Learned attention aggregator (lines 64–123)

### Lines 68–96 — Class header + `__init__`

```python
class _JKAttention(nn.Module):
```

Why is the leading underscore used? It's Python convention for a **private** class — not imported by `sentinel_model.py` directly, only used internally by `GNNEncoder`.

#### Design decision: Why implement JK ourselves instead of using PyG's?

The docstring (lines 76-78) gives two reasons:
1. **Explicit gradient flow** — `self.attn` is a single `Linear(ch, 1)` — trivial to verify it receives non-zero gradients (the gate `test_jk_gradient_flow`).
2. **No LSTM overhead** — PyG offers `JumpingKnowledge(mode='lstm')` which maintains hidden state across the sequence. We don't need that — we just need weighted averaging.

#### Three registered buffers (lines 88-96)

```python
self.register_buffer("last_weights", torch.zeros(num_phases))      # [3]
self.register_buffer("last_weight_stds", torch.zeros(num_phases))   # [3]
self.last_node_weights: torch.Tensor | None = None                   # [N, K] in eval only
```

**Why `register_buffer` vs plain tensor?** Buffers are automatically moved to the correct device by `.to()` and are included in `state_dict()`. They survive `model.cuda()`, `model.half()`, and DDP wrapping.

The three metrics serve:
- `last_weights` — mean attention per phase across ALL nodes. Tells you if the model relies equally on all phases.
- `last_weight_stds` — [A23] standard deviation per phase. **std ≈ 0 → attention is a global constant** — the model learned a fixed weight vector (e.g., [0.9, 0.05, 0.05]) and every node gets the same routing regardless of its type. std > 0.10 → the model is genuinely routing different nodes differently.
- `last_node_weights` — only populated in `eval()` mode. Used by `jk_weight_hist.py` diagnostic to create per-node-type histograms.

### Lines 98–123 — `forward()`

```python
def forward(self, xs: list[torch.Tensor]) -> tuple[torch.Tensor, torch.Tensor]:
    stacked = torch.stack(xs, dim=1)          # [N, K, channels] — stack 3 phase outputs
    scores  = self.attn(stacked)               # [N, K, 1] — Linear(ch, 1) maps each node's per-phase vector to a scalar
    weights = torch.softmax(scores, dim=1)     # [N, K, 1] — softmax normalises over phases
    w_nk    = weights.squeeze(-1)               # [N, K]
```

**The attention mechanism explained:**

For each node, we have 3 vectors (one per phase): `[v_p1, v_p2, v_p3]`. Each is 256-dim.

1. `self.attn(stacked)` computes `w^T · v_pi` for each phase — a single scalar "score" per phase.
2. `softmax` over phases gives a probability distribution: `[α_1, α_2, α_3]`, sum-to-1.
3. The output is `α_1·v_p1 + α_2·v_p2 + α_3·v_p3`.

If a node's structure is well-captured by Phase 1 (e.g., a STATE_VAR node), its α_1 will be high. If a node needs CFG context (e.g., a CFG_NODE_WRITE node in a CEI sequence), α_2 will be higher.

#### Buffer update — in-place with `.copy_()` (line 114)

```python
self.last_weights.copy_(w_nk.mean(0).detach())
self.last_weight_stds.copy_(w_nk.std(0, unbiased=False).nan_to_num(0.0).detach())
```

**`.copy_()` is required, not optional.** If you do `self.last_weights = ...` instead, you break the buffer registration — the new tensor is not tracked by `.to()` and `state_dict()`.

**`unbiased=False`** (line 115 comment, [A23]): When N=1 (single node in graph), `unbiased=True` divides by N-1 = 0 → NaN. `unbiased=False` divides by N → 0.0. The `.nan_to_num(0.0)` is a safety net for any edge case.

#### Per-node weight storage in eval mode (lines 117-118)

```python
if not self.training:
    self.last_node_weights = w_nk.detach().cpu()
```

Only stored in eval mode because the number of nodes (`N`) varies per batch. In training, `last_node_weights` stays `None`.

#### JK entropy (lines 120-122, C-3)

```python
jk_entropy = -(w_nk * (w_nk + 1e-8).log()).sum(dim=1).mean()
```

**Why do we compute this?** Entropy measures how "spread out" the attention distribution is over the 3 phases.

- `H = 0`: collapsed attention — one phase gets weight 1.0, others 0.0. No routing happening.
- `H = log(3) ≈ 1.099`: uniform attention — all phases equally weighted. Also no routing.
- `0 < H < log(3)`: the model is routing — different nodes attend to different phases.

**Why the `+ 1e-8`?** `log(0)` = -inf. If a phase gets 0.0 weight, the entropy breaks. Epsilon prevents this.

**Gradient attached:** The entropy is returned with gradients, so it can be used as an auxiliary loss (e.g., penalize entropy below 0.5 to prevent collapse).

---

## CHUNK 3: `GNNEncoder.__init__` — Headers + input processing (lines 130–237)

### Lines 130–154 — Class docstring

The docstring (lines 134-153) specifies:

```
Input:
    x:          node features       [N, NODE_FEATURE_DIM]
    edge_index: graph connectivity  [2, E]
    batch:      node→graph mapping  [N]
    edge_attr:  edge type IDs       [E]  int64 in [0, NUM_EDGE_TYPES)

Output (normal):
    node_embeddings: [N, hidden_dim]  — NOT pooled
    batch:           [N]              — passed through unchanged
    jk_entropy:      scalar

Alternate return flags:
    return_intermediates: also returns dict with per-phase detached tensors
    return_phase2_embs:   also returns Phase 2 output WITH gradients for aux loss
```

**Important: The GNN does NOT pool.** It returns per-node embeddings `[N, 256]`. Pooling happens in `SentinelModel` after the CrossAttentionFusion merges GNN outputs with token outputs.

### Lines 156–210 — `__init__` signature + architecture guard

```python
def __init__(
    self,
    hidden_dim:               int            = 256,
    heads:                    int            = 8,
    dropout:                  float          = 0.2,
    use_edge_attr:            bool           = True,
    edge_emb_dim:             int            = 64,
    num_layers:               int            = 8,
    use_jk:                   bool           = True,
    jk_mode:                  str            = 'attention',
    phase2_edge_types:        list[int]|None = None,
    validate_graph_integrity: bool           = False,  # [A25]
    drop_complexity:          bool           = False,  # Run 8
    appnp_alpha:              float          = 0.0,    # Run 8
) -> None:
```

**11 parameters** — this is the most complex `__init__` in the entire codebase. Let me trace how each is used:

| Parameter | Passed by | Effect |
|---|---|---|
| `hidden_dim` | `TrainConfig` | All GNN hidden dims (256) |
| `heads` | `TrainConfig` | Phase 1 heads only (8); Phase 2 uses 4, Phase 3 uses 1 regardless |
| `dropout` | `TrainConfig` | Dropout after every conv layer |
| `use_edge_attr` | `TrainConfig` | If False, no edge_embedding — edge types ignored |
| `edge_emb_dim` | `TrainConfig` | Dim of edge type embedding (64) |
| `num_layers` | validation only | MUST be 8 (see below) |
| `use_jk` | `TrainConfig` | If False, only Phase 3 output returned (v5 compat) |
| `jk_mode` | `TrainConfig` | Only 'attention' implemented |
| `phase2_edge_types` | `TrainConfig` | Which edge types to include in Phase 2 CFG mask |
| `validate_graph_integrity` | testing only | [A25] expensive O(E) check every forward |
| `drop_complexity` | experiment | [Run 8] zero feat[5] |
| `appnp_alpha` | experiment | [Run 8] APPNP teleport fraction |

#### Architecture lock (lines 191-198, [A27])

```python
if num_layers != SENTINEL_GNN_NUM_LAYERS:
    raise ValueError(...)
```

Any attempt to construct with `num_layers=4` or `num_layers=12` fails hard. The architecture is not parameterized by layer count — it's a fixed 2+3+3 design. If you want a 12-layer GNN, you need to re-design the phases.

#### Head dimension divisibility (lines 211-216)

```python
_head_dim = hidden_dim // heads  # 32 per head when hidden=256, heads=8
if _head_dim * heads != hidden_dim:
    raise ValueError(...)
```

This isn't just a nice-to-have — PyG's GATConv does NOT check this. If `hidden_dim=257` and `heads=8`, you get `257//8 = 32`, `32*8 = 256`, but `concat=True` produces output 256, not 257. The error would be a shape mismatch when connecting to the next layer.

### Lines 218–237 — Embedding layers + skip projection

#### Edge embedding (lines 218-225)

```python
_edge_dim: int | None = None
if use_edge_attr:
    self.edge_embedding = nn.Embedding(NUM_EDGE_TYPES, edge_emb_dim)
    _edge_dim = edge_emb_dim
```

**`NUM_EDGE_TYPES = 12`** (IDs 0-11). This means the embedding table has 12 rows, each 64-dim. When GATConv receives `edge_dim=_edge_dim`, it concatenates the edge embedding to the node feature during attention computation: `a(W·h_i || W·h_j || W_e·e_ij)`.

Edge type IDs on disk are 0-11. Type 7 (REVERSE_CONTAINS) is runtime-only — constructed during `forward()`, not stored in `.pt` files. It uses the same embedding table but with ID 7.

#### Type embedding (lines 227-231, BUG-R7-2)

```python
self.type_embedding = nn.Embedding(_NUM_NODE_TYPES, _TYPE_EMB_DIM)
```

`_NUM_NODE_TYPES = 14`, `_TYPE_EMB_DIM = 16`. Table size: `14 × 16 = 224 parameters` — negligible but critical.

**Why not one-hot encode?** One-hot encoding would produce a 14-dim vector. We use 16-dim because it gives the embedding head more capacity to learn similarity structure (e.g., FUNCTION nodes similar to MODIFIER nodes but different from CFG_NODE_CALL).

#### Input projection skip (lines 233-237, IMP-G2)

```python
self.input_proj = nn.Linear(_GNN_IN_DIM, hidden_dim, bias=False)
```

**Why `bias=False`?** The conv layer has its own bias. Adding a second bias would create a redundant degree of freedom.

**Why 6,912 parameters?** `28 × 256 = 7,168` without bias. The docstring says 6,912 — wait, that's 27 × 256. The docstring says `_GNN_IN_DIM=27` but the code says `_GNN_IN_DIM=28` (12 + 16). This means the docstring is STALE — it was written when `_GNN_IN_DIM` was 27. The code is the source of truth: 28 × 256 = 7,168 params.

**The purpose of the skip (from docstring line 12):** GATConv starts with near-uniform attention weights. Without the skip, the first conv layer could lose raw feature information because every neighbor gets equal weight. The skip ensures that the raw feature information is always present after conv1's output.

---

## CHUNK 4: `GNNEncoder.__init__` — Layer definitions (lines 239–358)

### Lines 239–257 — Phase 1: Structural (2 layers, 8 heads, self-loops)

```python
self.conv1 = GATConv(
    in_channels=_GNN_IN_DIM,   # 28
    out_channels=_head_dim,     # 32 per head
    heads=heads,                # 8 → total output = 8×32 = 256
    concat=True,
    add_self_loops=True,
    edge_dim=_edge_dim,         # 64
)
self.conv2 = GATConv(
    in_channels=hidden_dim,     # 256 (output of conv1)
    out_channels=_head_dim,     # 32 per head → 256 total
    heads=heads,
    concat=True,
    add_self_loops=True,
    edge_dim=_edge_dim,
)
```

**Phase 1 preserves all edges types 0-5.** These are the "declaration-level" edges: CALLS, READS, WRITES, EMITS, INHERITS, CONTAINS. These describe which functions call which, which state variables are read/written, and the contract-function hierarchy.

**`add_self_loops=True`** — each node attends to itself. This means even an ISOLATED node (no neighbours matching struct_mask) will get its own features processed through the GAT mechanism. In Phase 2/3, self-loops are harmful (they dilute directional signal), but in Phase 1 they're essential because not every node participates in CALLS/READS/WRITES patterns.

**Why 8 heads?** More heads = more attention patterns. Each of the 8 heads can learn to attend to different structural relationships: one head for CALLS patterns, one for READS patterns, one for WRITES patterns, etc.

### Lines 259–299 — Phase 2: CFG + ICFG (3 layers, 4 heads, NO self-loops)

```python
_p2_heads    = 4
_p2_head_dim = hidden_dim // _p2_heads   # 64 per head when hidden=256

self.conv3  = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
self.conv3b = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
self.conv3c = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
```

**All three Phase 2 convs have identical construction** — the same `hidden_dim→64` with 4 heads. The DIFFERENCE is which edges they receive (IMP-G1), which is applied at forward time, not init time.

**IMP-R7-1 (line 262-263 comments):** Originally Phase 2 used 1 head at 256-dim. With 1 head, the single attention function must capture ALL CFG patterns. With 4 heads at 64-dim each, heads can specialise:
- Head 1: **write-order** — a WRITE after a CALL (CEI violation)
- Head 2: **call-depth** — how deep in the call chain is this node
- Head 3: **CEI sequence** — CHECK before effect
- Head 4: **data-flow timing** — when state is read vs written

**Parameter count is identical** to the old design: 1 head at `256→256` uses `256*256 + 256(bias) = 65,792` params. 4 heads at `256→64` each uses `4*(256*64 + 64) = 4*16,448 = 65,792`. Same params, different distribution.

**`add_self_loops=False` is the most critical single flag** in the GNN (explicitly stated as "CRITICAL" in the comments). Here's why:

In CFG message passing, you want information to flow FROM predecessors TO successors:
```
CALL_node → (CF edge) → TMP_node → (CF edge) → WRITE_node
```

With `add_self_loops=True`:
```
CALL_node → itself (self-loop)
CALL_node → TMP_node
TMP_node → itself  (self-loop)
TMP_node → WRITE_node
```

The self-loop means each node's own representation contributes 50% (or more, depending on attention weight) of its update. This dilutes the predecessor→successor signal so much that after 3 hops, the CEI pattern is invisible.

**BUG-H1 (lines 287-291, `conv3c`):** The third CF layer (Layer 5) was added to capture the full ENTRY→CALL→TMP→WRITE sequence. With only 2 hops (conv3+conv3b), the model could see CALL→TMP→WRITE (the classic 2-step CEI pattern) but missed the function ENTRY that set up the call. 3 hops covers ENTRY→CALL→TMP→WRITE — the complete CEI sequence from function entry to state write.

### Lines 301–331 — Phase 3: Bidirectional CONTAINS (3 layers, 1 head, NO concat)

```python
self.conv4   = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
self.conv4b  = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
self.conv4c  = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
```

**`heads=1, concat=False`** — these layers preserve the hidden_dim (no expansion/contraction). This is because Phase 3 aggregates across the CONTAINS hierarchy without changing representation size.

**Why only 1 head?** Phase 3 has one job: move information up and down the CONTAINS tree. You don't need multiple attention patterns for this — every edge IS a CONTAINS relationship. The only question is "how much information flows across each edge."

**IMP-G3 (line 323):** The downward pass (`conv4c`) was added so that FUNCTION nodes distribute their enriched context back to CFG children. Without this, after 2 upward hops, FUNCTION nodes would have richer embeddings than CFG nodes, and `CrossAttentionFusion` would produce asymmetrical outputs.

### Lines 333–358 — LayerNorm, JK selector, dropout, dtype cache

#### Per-phase LayerNorm (lines 333-340, Phase 1-A2)

```python
self.phase_norm = nn.ModuleList([
    nn.LayerNorm(hidden_dim),  # after Phase 1
    nn.LayerNorm(hidden_dim),  # after Phase 2
    nn.LayerNorm(hidden_dim),  # after Phase 3
])
```

**Why LayerNorm after each phase, not after each layer?** JK aggregation is sensitive to scale differences. If Phase 1 outputs have norm 100 and Phase 2 outputs have norm 5, Phase 1 will dominate the JK softmax regardless of content. LayerNorm normalises each phase's output to zero-mean/unit-variance before JK sees them.

Three separate `nn.LayerNorm` instances (one per phase, in `ModuleList`) so each phase learns its own scale/gain parameters.

#### JK selector (lines 342-351)

```python
if use_jk:
    if jk_mode != 'attention':
        raise ValueError("Only jk_mode='attention' is implemented.")
    self.jk = _JKAttention(hidden_dim)
else:
    self.jk = None
```

JK can be disabled (`use_jk=False`) for checkpoint compatibility with v5 models that didn't have JK. Without JK, only Phase 3 output is returned.

#### Dtype cache (lines 356-358, [A26])

```python
self._param_dtype: torch.dtype = next(self.parameters()).dtype
```

**Why not `next(self.parameters()).dtype` in forward?** Because calling `next(self.parameters())` iterates over ALL 2.5M parameters just to get the dtype. Caching it saves O(2.5M) iterator overhead every forward pass.

**`refresh_dtype_cache()`** (lines 360-362): After any runtime dtype cast (`.float()`, `.half()`, `.bfloat16()`), the cache becomes stale. Call `model.gnn.refresh_dtype_cache()` after casting.

---

## CHUNK 5: `GNNEncoder.forward` — Guards, embeddings, masks (lines 364–568)

### Lines 364–397 — Method signature and docstring

```python
def forward(
    self,
    x:                    torch.Tensor,
    edge_index:           torch.Tensor,
    batch:                torch.Tensor,
    edge_attr:            torch.Tensor | None = None,
    return_intermediates: bool               = False,
    return_phase2_embs:   bool               = False,
):
```

**Two special return modes:**
- `return_intermediates=True`: returns `(emb, batch, entropy, dict)` — dict has detached per-phase outputs for diagnostic use (e.g., `gnn_intermediate_diagnostics.py`).
- `return_phase2_embs=True`: returns `(emb, batch, entropy, phase2_x)` — phase2_x has GRADIENTS attached, for the CEI auxiliary loss. Since `return_intermediates` uses `.detach().clone()`, the two modes are mutually exclusive (caught upstream in `SentinelModel`).

### Lines 398–428 — Input guards

#### Guard 1: Feature dimension mismatch (lines 399-403)

```python
if x.shape[1] != NODE_FEATURE_DIM:
    raise ValueError(...)
```

**When does this trigger?** If someone loads a stale `.pt` file from schema v6 (which had `NODE_FEATURE_DIM=11`). The error tells you to re-run `reextract_graphs.py` instead of getting a silent shape mismatch inside GATConv.

#### Guard 2: `use_edge_attr=True` but `edge_attr=None` (lines 407-412)

```python
if self.use_edge_attr and edge_attr is None:
    raise ValueError(...)
```

**Bug #1:** Without edge_attr, the `cfg_mask` becomes all-zeros (lines 500-501), which means Phase 2 receives NO edges and produces zero output. The model silently degrades to a 5-layer GNN. This guard catches the bug early.

#### Guard 3: OOB node indices (lines 414-422, [A25])

```python
if self.validate_graph_integrity and edge_index.numel() > 0 and edge_index.max() >= x.shape[0]:
    raise ValueError(...)
```

**`validate_graph_integrity=False` by default** because `edge_index.max()` scans all E edges — O(E) on every forward pass. For a 1,000-node graph with 5,000 edges, that's 5,000 comparisons per training step. Enable only for debugging.

**What causes OOB indices?** A corrupted `.pt` file where the edge index references node IDs that don't exist. Without this guard, GATConv silently uses garbage attention weights or crashes with CUDA illegal memory access.

#### Dtype normalisation (lines 424-429, [A26])

```python
if x.dtype != self._param_dtype:
    x = x.to(self._param_dtype)
```

**When does this matter?** CodeBERT loads in BF16 (bfloat16). If the model's GNN is float32 and BERT sends BF16 activations, the dtypes don't match. This guard silently casts.

### Lines 431–437 — Complexity-proxy break (Run 8)

```python
if self.drop_complexity:
    x = x.clone()
    x[:, 5] = 0.0
```

**`.clone()` is mandatory.** Without it, `x[:, 5] = 0.0` would modify the tensor IN PLACE. Since `x` is a view of the batch tensor in `SentinelDataset`, this would corrupt the cached graph tensor and affect all future batches that reference the same cached entry.

**Why feat[5]?** Complexity (cyclomatic complexity) is a strong proxy for vulnerability: big contracts have more bugs. The L4 experiment showed that the model was using complexity as a shortcut (high complexity → always predict several vulnerabilities). Zeroing it forces the model to learn actual semantic patterns.

### Lines 439–459 — Edge embeddings with OOB clamping (Fix C1)

```python
if self.edge_embedding is not None and edge_attr is not None:
    if edge_attr.numel() > 0:
        _max_valid = self.edge_embedding.num_embeddings - 1   # 11
        _oob_mask = (edge_attr < 0) | (edge_attr > _max_valid)
        if _oob_mask.any():
            logger.warning(...)
            edge_attr = edge_attr.clamp(0, _max_valid)
    e = self.edge_embedding(edge_attr)   # [E, edge_emb_dim]
```

**Fix C1 (H9):** Corrupted `.pt` files can contain edge type IDs of 999 or -1. `nn.Embedding` maps these to random embeddings (for positive OOB) or crashes (for negative indices). Clamping ensures the forward pass continues with a logged warning instead of killing the entire training run with a CUDA context loss.

**Why only a warning and not an error?** The model can tolerate a few corrupted edges — they'll get mapped to the nearest valid embedding. But an uncaught CUDA error destroys the training process and loses all in-flight gradients.

### Lines 461–506 — Edge mask construction

#### Edge type constants (lines 466-472)

```python
_CONTAINS         = EDGE_TYPES["CONTAINS"]           # 5
_CONTROL_FLOW     = EDGE_TYPES["CONTROL_FLOW"]        # 6
_REVERSE_CONTAINS = EDGE_TYPES["REVERSE_CONTAINS"]    # 7 (runtime-only)
_CALL_ENTRY       = EDGE_TYPES["CALL_ENTRY"]          # 8
_RETURN_TO        = EDGE_TYPES["RETURN_TO"]           # 9
_DEF_USE          = EDGE_TYPES["DEF_USE"]             # 10
_EXTERNAL_CALL    = EDGE_TYPES.get("EXTERNAL_CALL", -1)  # 11 (v9 Fix #3)
```

**`.get("EXTERNAL_CALL", -1)` — the fallback to -1 is important.** If the `EDGE_TYPES` dict doesn't have `EXTERNAL_CALL` (because someone loaded an old `graph_schema.py`), it defaults to -1. Since no edge has type -1, `EXTERNAL_CALL` simply doesn't participate in any mask — it's silently excluded instead of crashing.

#### Structural mask (line 474)

```python
struct_mask = edge_attr <= _CONTAINS
```

**Every edge type from 0 to 5 goes to Phase 1.** This includes CALLS, READS, WRITES, EMITS, INHERITS, and CONTAINS. Type 6+ are CFG/ICFG edges for Phase 2.

#### CFG mask with ablation support (lines 475-486)

```python
if self.phase2_edge_types is not None:
    cfg_mask = torch.zeros(...)
    for _t in self.phase2_edge_types:
        cfg_mask |= (edge_attr == _t)
else:
    cfg_mask = (
        (edge_attr == _CONTROL_FLOW) |
        (edge_attr == _CALL_ENTRY)   |
        (edge_attr == _RETURN_TO)    |
        (edge_attr == _DEF_USE)      |
        (edge_attr == _EXTERNAL_CALL)
    )
```

**Ablation support:** If `phase2_edge_types=[6, 8, 9]`, only CONTROL_FLOW, CALL_ENTRY, and RETURN_TO participate in Phase 2. DEF_USE and EXTERNAL_CALL are excluded. This allows experiments that test "do we actually need DEF_USE for CFG routing?"

#### Edge_attr=None degraded mode (lines 488-501)

```python
else:
    n_edges = edge_index.shape[1]
    struct_mask   = torch.ones(n_edges, dtype=torch.bool, ...)
    cfg_mask      = torch.zeros(n_edges, dtype=torch.bool, ...)
    contains_mask = torch.zeros(n_edges, dtype=torch.bool, ...)
```

When `edge_attr is None`,:
- `struct_mask = all True` — ALL edges participate in Phase 1
- `cfg_mask = all False` — Phase 2 receives NO edges (zero output)
- `contains_mask = all False` — Phase 3 receives NO edges (zero output)

The model still works — it becomes a 2-layer GNN on all edges. But Phase 2 and Phase 3 outputs are zero, which means JK will give them zero weight after softmax stabilizes. The warning is logged to alert the data pipeline team.

### Lines 503–541 — Phase 2 sub-masks (IMP-G1, NF-6)

```python
struct_ei = edge_index[:, struct_mask]
struct_ea = e[struct_mask]   if e is not None else None

phase2_ei    = edge_index[:, cfg_mask]
phase2_ea    = e[cfg_mask]      if e is not None else None
```

**These create the base edge subsets for each phase.**

Then Phase 2 is further subdivided (lines 519-540):

```python
phase2_raw = edge_attr[cfg_mask]       # [E_cfg] raw int type IDs
_cf_mask   = (phase2_raw == _CONTROL_FLOW)
_icfg_mask = (
    (phase2_raw == _CALL_ENTRY) |
    (phase2_raw == _RETURN_TO)  |
    (phase2_raw == _EXTERNAL_CALL)      # v9 Fix #3
)
cf_only_ei     = phase2_ei[:, _cf_mask]
cf_only_ea     = phase2_ea[_cf_mask]
icfg_only_ei   = phase2_ei[:, _icfg_mask]
icfg_only_ea   = phase2_ea[_icfg_mask]
```

**Why sub-masks on already-masked data (NF-6)?** Before NF-6 fix, `conv3` and `conv3b` read from the full `edge_index` (not from the already-ablated `phase2_ei`). This meant that `--phase2-edge-types` ablation — which was SUPPOSED to exclude certain edge types — only affected `conv3c` (the integration layer) but not `conv3` and `conv3b`.

**v9 Fix #3 (line 525):** EXTERNAL_CALL edges are routed through conv3b (the ICFG cross-function layer) because they represent calls to external contracts — semantically aligned with CALL_ENTRY (entering another function) and RETURN_TO (returning from another function).

### Lines 542–568 — Phase 3 reverse edge construction (Phase 1-A3)

```python
fwd_contains_ei = edge_index[:, contains_mask]           # [2, E_contains]
fwd_contains_ea = e[contains_mask] if e is not None else None
rev_contains_ei = fwd_contains_ei.flip(0)                # [2, E_contains]
```

**`.flip(0)` reverses the edge direction.** If CONTAINS goes `CONTRACT→FUNCTION→CFG_NODE`, `.flip(0)` makes it `CFG_NODE→FUNCTION→CONTRACT`. This is how upward message passing works — we reverse the CONTAINS edges.

**Type 7 edge embedding construction (lines 550-561):**

```python
if self.edge_embedding is not None:
    n_rev = rev_contains_ei.shape[1]
    if n_rev > 0:
        rev_type_ids = torch.full((n_rev,), _REVERSE_CONTAINS, dtype=torch.long, ...)
        rev_contains_ea = self.edge_embedding(rev_type_ids)  # [E_contains, edge_emb_dim]
    else:
        rev_contains_ea = None
```

**REVERSE_CONTAINS (type 7) is a runtime-only edge type.** It's NEVER stored on disk. Every forward pass creates fresh type-7 edge embeddings for the reverse edges. This uses the same `edge_embedding` table — row 7 — just with a different type ID.

**Why not store REVERSE_CONTAINS on disk?** Storage efficiency. The forward CONTAINS edges represent parent→child relationships in the AST. The reverse is just `edge_index.flip(0)` — you can derive it from the forward direction at runtime. Saving both would double CONTAINS storage for no new information.

### Lines 563–568 — Live list + intermediates dict

```python
_live: list[torch.Tensor] = []
_intermediates: dict      = {}
```

**`_live`** collects phase outputs WITH gradients attached (for JK attention parameter training).
**`_intermediates`** collects `.detach().clone()` copies for diagnostics.

**CRITICAL: Do NOT put `.detach()` into `_live`.** The JK attention layer needs gradients flowing from the final loss back through each phase output. If you detach, `self.jk.attn` never trains.

---

## CHUNK 6: `GNNEncoder.forward` — Phase 1 + Phase 2 (lines 570–631)

### Lines 570–593 — Phase 1: Structural aggregation

#### Type embedding concatenation (lines 574-575, BUG-R7-2)

```python
_type_int = (x[:, 0].float() * _NUM_NODE_TYPES).round().long().clamp(0, _NUM_NODE_TYPES - 1)
x_init = torch.cat([x, self.type_embedding(_type_int).to(x.dtype)], dim=1)  # [N, 28]
```

**Decoding the type ID:** feat[0] stores `type_id / 12.0`. To recover the integer:
1. Multiply by `_NUM_NODE_TYPES` (14): `x[:, 0] * 14`
2. Round to nearest integer (handles floating point fuzziness)
3. Convert to long (for Embedding lookup)
4. Clamp to [0, 13] (safety — if rounding gives 14 for some edge case)

**Result:** `x_init` is `[N, 12 + 16] = [N, 28]` — the original features plus a learned 16-dim type representation.

#### Skip connection (lines 578-583, IMP-G2)

```python
x_skip = self.input_proj(x_init.to(self._param_dtype)).to(x_init.dtype)  # dtype-safe
x  = self.conv1(x_init, struct_ei, struct_ea)   # [N, 28] → [N, 256]
x  = self.relu(x + x_skip)                       # skip added before relu
x  = self.dropout(x)
```

**The skip is added BEFORE ReLU.** This is critical — if the skip were added after ReLU, negative values from the conv output would zero out the skip's contribution.

**Why dtype-safe cast?** `x_init` might be in the batch's dtype (e.g., BF16 from BERT-dominant pipeline), but `input_proj` weights are in the model's dtype. The `.to(self._param_dtype)` ensures correct matmul, then `.to(x_init.dtype)` ensures the output matches the activation dtype for downstream ops.

#### Residual connection (lines 586-588)

```python
x2 = self.conv2(x, struct_ei, struct_ea)    # [N, 256] → [N, 256]
x2 = self.relu(x2)
x  = x + self.dropout(x2)                   # residual: identity preserved, only branch dropped
```

**Standard pre-activation residual:** The dropout is applied ONLY to the branch, not the identity. This means at inference (dropout disabled), the residual connection is exact: `x = x + x2`. During training, it degrades the branch by the dropout rate.

#### Phase 1 LayerNorm (line 590, Phase 1-A2)

```python
x  = self.phase_norm[0](x)
_live.append(x)
_intermediates["after_phase1"] = x.detach().clone()
```

**`_live` gets the gradient-attached tensor.** `_intermediates` gets a snapshot for diagnostics.

### Lines 595–631 — Phase 2: CFG + ICFG directed

#### APPNP anchor setup (lines 610)

```python
_phase1_anchor = x.detach() if self.appnp_alpha > 0.0 else None
```

**`.detach()` is critical here.** If we didn't detach, the Phase 2 layers would have a gradient shortcut back to Phase 1. Phase 1 gradients should only flow through JK aggregation, NOT through the APPNP teleport. Without detach, Phase 1 would get double gradients: once via JK and once via each Phase 2 APPNP blend.

#### Layer 3 — CONTROL_FLOW only (conv3, lines 612-616)

```python
x2 = self.conv3(x, cf_only_ei, cf_only_ea)        # CF only edges
x2 = self.relu(x2)
x  = x + self.dropout(x2)
if _phase1_anchor is not None:
    x = self.appnp_alpha * _phase1_anchor + (1.0 - self.appnp_alpha) * x
```

**Only CONTROL_FLOW edges participate.** This lets the layer focus purely on intra-function execution ordering without ICFG noise. CEI patterns begin to form here as CALL→TMP→WRITE edges.

**APPNP teleport (Run 8):** Without teleport, each CFG hop dilutes the Phase 1 signal. After 3 hops with avg degree=5: `signal × 0.2^3 = 0.008` — less than 1% of original. The teleport ensures the structural signal is always at least `α * 100%` present.

#### Layer 4 — CALL_ENTRY + RETURN_TO only (conv3b, lines 617-621)

```python
x2 = self.conv3b(x, icfg_only_ei, icfg_only_ea)     # ICFG only edges
x2 = self.relu(x2)
x  = x + self.dropout(x2)
if _phase1_anchor is not None:
    x = self.appnp_alpha * _phase1_anchor + (1.0 - self.appnp_alpha) * x
```

**ICFG (Interprocedural CFG):** CALL_ENTRY edges go from a CALL node to the FUNCTION it calls. RETURN_TO edges go back. This layer builds cross-function call context — a CALL in function A now carries information about function B's state accesses.

#### Layer 5 — Joint integration (conv3c, lines 622-626)

```python
x2 = self.conv3c(x, phase2_ei, phase2_ea)           # All Phase 2 edges
x2 = self.relu(x2)
x  = x + self.dropout(x2)
if _phase1_anchor is not None:
    x = self.appnp_alpha * _phase1_anchor + (1.0 - self.appnp_alpha) * x
```

**This layer receives ALL Phase 2 edge types** (CF + CALL_ENTRY + RETURN_TO + DEF_USE + EXTERNAL_CALL). It integrates the specialized patterns from layers 3 and 4 into a unified CFG/ICFG representation.

#### Phase 2 LayerNorm + capture (lines 627-631)

```python
x  = self.phase_norm[1](x)
_live.append(x)
_intermediates["after_phase2"] = x.detach().clone()
_phase2_x = x  # gradient-attached reference for aux loss
```

**`_phase2_x` is saved separately** (not just through `_live`, which goes to JK). This is for the CEI auxiliary loss in `SentinelModel`, which needs Phase 2's output specifically (not JK-aggregated).

---

## CHUNK 7: `GNNEncoder.forward` — Phase 3, JK, return (lines 633–667)

### Lines 633–652 — Phase 3: Bidirectional CONTAINS

#### Layer 6 — First upward hop (conv4, lines 639-641)

```python
x2 = self.conv4(x, rev_contains_ei, rev_contains_ea)
x2 = self.relu(x2)
x  = x + self.dropout(x2)
```

**`rev_contains_ei`** is the reversed CONTAINS edges: `CFG_NODE → FUNCTION → CONTRACT`. Phase 2 signal (which is on CFG nodes) now flows UP to FUNCTION nodes.

**Why only one head?** This is pure aggregation — bring information from many CFG children to their parent FUNCTION. You don't need multiple attention patterns for "how much of each child's info flows up."

#### Layer 7 — Second upward hop (conv4b, lines 642-644)

```python
x2 = self.conv4b(x, rev_contains_ei, rev_contains_ea)
x2 = self.relu(x2)
x  = x + self.dropout(x2)
```

**Why two hops up?** One hop only reaches the immediate parent FUNCTION. The second hop lets a CFG node in one function's subtree influence sibling functions. Consider:

```
CONTRACT
├── FUNCTION_A
│   ├── CFG_NODE_CALL → calls FUNCTION_B
│   └── CFG_NODE_WRITE
└── FUNCTION_B
    └── CFG_NODE_WRITE
```

After 1 upward hop, FUNCTION_A knows about its own CFG nodes. After 2 upward hops, CONTRACT knows about both FUNCTION_A and FUNCTION_B. Then the downward pass (conv4c) distributes this contract-wide context back to all CFG nodes.

#### Layer 8 — Downward pass (conv4c, lines 645-648, IMP-G3)

```python
x2 = self.conv4c(x, fwd_contains_ei, fwd_contains_ea)
x2 = self.relu(x2)
x  = x + self.dropout(x2)
```

**`fwd_contains_ei`** uses the ORIGINAL CONTAINS direction: `CONTRACT → FUNCTION → CFG_NODE`. This carries the contract-wide enriched context DOWN to every CFG node.

**Why this matters for CrossAttentionFusion:** After the two upward hops, FUNCTION nodes have 2 extra aggregation steps compared to CFG nodes. Without the downward pass, the GNN eye would output FUNCTION nodes with richer embeddings than CFG nodes. The `CrossAttentionFusion` layer attends tokens to GNN nodes — if all nodes aren't equally enriched, the fusion is suboptimal.

#### Phase 3 LayerNorm (line 649)

```python
x  = self.phase_norm[2](x)
_live.append(x)
_intermediates["after_phase3"] = x.detach().clone()
```

### Lines 654–667 — JK aggregation + return

#### JK aggregation (lines 654-661, Phase 1-A1)

```python
if self.use_jk and self.jk is not None:
    x, _jk_entropy = self.jk(_live)
else:
    _jk_entropy = x.new_zeros(1).squeeze()
```

**`_jk_entropy` is a zero scalar when JK is disabled** — it has no gradients, no effect, and doesn't participate in the aux loss.

**Three return paths:**

```python
if return_intermediates:
    return x, batch, _jk_entropy, _intermediates      # 4 outputs
if return_phase2_embs:
    return x, batch, _jk_entropy, _phase2_x            # 4 outputs
return x, batch, _jk_entropy                           # 3 outputs
```

**The 3-output vs 4-output difference matters** because `SentinelModel.forward()` checks the length of the return tuple to determine which return mode was used.

---

## Appendix A: Complete forward pass call graph

```
GNNEncoder.forward(x, edge_index, batch, edge_attr)
│
├── Guards: feature dim, edge_attr None, graph integrity, dtype
│
├── Optional: drop_complexity (zero feat[5])
│
├── Edge embeddings: OOB clamp → nn.Embedding → [E, 64]
│
├── Edge masks: struct_mask, cfg_mask, contains_mask
│   ├── Phase 2 sub-masks: cf_mask, icfg_mask
│   └── Phase 3: fwd_contains + rev_contains (type-7 runtime)
│
├── Phase 1 (structural, layers 1-2)
│   ├── Type embedding enrichment: [N, 12] → [N, 28]
│   ├── conv1 + input_proj skip (IMP-G2) → [N, 256]
│   └── conv2 + residual → [N, 256] → LayerNorm → _live[0]
│
├── Phase 2 (CFG/ICFG, layers 3-5)
│   ├── conv3 (CF only) + APPNP teleport
│   ├── conv3b (ICFG only) + APPNP teleport
│   ├── conv3c (joint) + APPNP teleport
│   └── LayerNorm → _live[1]
│
├── Phase 3 (bidir CONTAINS, layers 6-8)
│   ├── conv4 (up, rev_contains)
│   ├── conv4b (up, rev_contains)
│   ├── conv4c (down, fwd_contains)  ← IMP-G3
│   └── LayerNorm → _live[2]
│
├── JK: _JKAttention(_live) → weighted sum + entropy
│
└── Return: [N, 256], batch, entropy (+ intermediates or phase2_embs)
```

## Appendix B: Parameter count breakdown

| Component | Details | Params |
|---|---|---|
| `type_embedding` | 14 × 16 | 224 |
| `input_proj` | 28 × 256 | 7,168 |
| `edge_embedding` | 12 × 64 | 768 |
| `conv1` | GATConv(28→32, H=8, edge_dim=64) | ~36K |
| `conv2` | GATConv(256→32, H=8, edge_dim=64) | ~168K |
| `conv3` | GATConv(256→64, H=4, edge_dim=64) | ~70K |
| `conv3b` | GATConv(256→64, H=4, edge_dim=64) | ~70K |
| `conv3c` | GATConv(256→64, H=4, edge_dim=64) | ~70K |
| `conv4` | GATConv(256→256, H=1, edge_dim=64) | ~84K |
| `conv4b` | GATConv(256→256, H=1, edge_dim=64) | ~84K |
| `conv4c` | GATConv(256→256, H=1, edge_dim=64) | ~84K |
| `phase_norm` | 3 × LayerNorm(256) = 3 × 512 | 1,536 |
| `jk.attn` | Linear(256, 1) | 256 |
| **Total** | | **~2.5M** |

(**Note:** PyG's GATConv parameter count varies — it includes attention vectors `a` per head plus the linear projection. The above is approximate.)

## Appendix C: Bug and ticket reference index

| ID | Lines | Description |
|---|---|---|
| BUG-R7-2 | 54-57, 227-231, 574-575 | Continuous type_id scalar → need learned embedding |
| IMP-G1 | 18-20, 509-540 | Each Phase 2 layer gets distinct edge subset |
| IMP-G2 | 12, 233-237, 578-583 | Input skip from raw features to conv1 output |
| IMP-G3 | 23-24, 323-331, 645-648 | Downward CONTAINS after upward |
| IMP-R7-1 | 16, 262-267, 271-298 | Phase 2 heads=4 (was 1) |
| Phase 1-A1 | 80-81, 95, 103-123 | JK attention aggregator |
| Phase 1-A2 | 29, 333-340, 590, 627, 649 | Per-phase LayerNorm |
| Phase 1-A3 | 543-561, 556 | REVERSE_CONTAINS type-7 embeddings |
| A23 | 115 | unbiased=False for std when N=1 |
| A25 | 167, 414-422 | validate_graph_integrity guard |
| A26 | 356-362, 424-429 | Dtype cache + normalisation |
| A27 | 61-62, 191-198 | Architecture lock at 8 layers |
| BUG-H1 | 287-291, 292-299 | conv3c — 3rd CF hop for full CEI |
| Fix C1 | 442-458, 448-458 | OOB edge_attr clamping |
| C-3 | 120-122 | JK entropy computation |
| NF-6 | 516-517, 519-540 | Phase 2 sub-masks on already-ablated data |
| Run 8 | 168-169, 208-209, 431-437 | Complexity-proxy break + APPNP |
