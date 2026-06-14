# ML02 — GNN Encoder: Three-Phase, 8-Layer GAT

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML02 of 10
**Source:** `ml/src/models/gnn_encoder.py` (667L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Graph schema** (Session 15): 13 node types, 11 edge types, 11 node features
- **CFG + call graph** (Session 18): CONTROL_FLOW, CALL_ENTRY, RETURN_TO edges
- **Four-eye model** (ML01): GNN eye uses output of this encoder

---

## §1 Three-Phase Design (2+3+3 = 8 layers)

```
Phase 1 (L1+L2): Structural aggregation + input skip
  Edges: CALLS(0)–CONTAINS(5), add_self_loops=True, heads=8
  Purpose: Build basic node representations from the raw graph

Phase 2 (L3+L4+L5): CFG + ICFG directed
  Edges: CONTROL_FLOW(6), CALL_ENTRY(8), RETURN_TO(9), DEF_USE(10)
  add_self_loops=False (CRITICAL), heads=4
  Purpose: Intra-function execution ordering + cross-function call structure

Phase 3 (L6+L7+L8): Bidirectional CONTAINS
  Edges: REVERSE_CONTAINS up, then CONTAINS down
  heads=1, concat=False
  Purpose: CFG signal rises to FUNCTION nodes, then enriched context flows back
```

---

## §2 Phase Details

### Phase 1 — Structural (lines 239–257)

```python
self.conv1 = GATConv(_GNN_IN_DIM, _head_dim, heads=8, concat=True, add_self_loops=True)
self.conv2 = GATConv(hidden_dim, _head_dim, heads=8, concat=True, add_self_loops=True)
```

Input: `_GNN_IN_DIM = 28` (11 features + 16-dim type embedding + 1).
Every edge type 0–5 participates. Self-loops ensure base features flow even without neighbors.

**IMP-G2: Input projection skip** (line 237):
```python
self.input_proj = nn.Linear(_GNN_IN_DIM, hidden_dim, bias=False)  # 6,912 params
```
Prevents raw feature loss in early training when GAT attention weights start near-uniform. Added to conv1 output before ReLU.

### Phase 2 — CFG (lines 259–299)

```python
self.conv3  = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
self.conv3b = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
self.conv3c = GATConv(hidden_dim, 64, heads=4, concat=True, add_self_loops=False)
```

**IMP-R7-1:** heads=4 (was 1 in v7). Each head specialises: write-order, call-depth, CEI sequence, data-flow timing. Parameter count is same as 1 head at 256-dim — FLOPS split across 4 heads.

**add_self_loops=False is CRITICAL** — self-loops cancel the directional CFG signal. Without this, a node attending to itself would dilute its predecessor's influence.

**Edge masking** (lines 461–486):
```python
cfg_mask = (edge_attr == CONTROL_FLOW) | (edge_attr == CALL_ENTRY) | ...
```
Each Phase 2 layer receives a different edge subset per IMP-G1:
- Layer 3 (conv3): CONTROL_FLOW(6) only — intra-function execution ordering
- Layer 4 (conv3b): CALL_ENTRY(8) + RETURN_TO(9) only — cross-function calls
- Layer 5 (conv3c): All Phase 2 types — joint integration

### Phase 3 — Bidirectional CONTAINS (lines 301–331)

```python
self.conv4   = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
self.conv4b  = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
self.conv4c  = GATConv(hidden_dim, hidden_dim, heads=1, concat=False)
```

**IMP-G3:** Two upward hops (REVERSE_CONTAINS), one downward hop (CONTAINS).
- Layer 6: CFG node signal rises to its parent FUNCTION node
- Layer 7: Second hop — multi-function patterns across sibling CFG subtrees
- Layer 8: Enriched FUNCTION context flows back down to CFG children

**Why bidirectional?** CrossAttentionFusion needs equally-enriched embeddings from ALL node types. Without downward propagation, FUNCTION nodes would have 2 extra hops of enrichment vs. CFG nodes.

---

## §3 JK Attention Aggregator (lines 68–123)

```python
class _JKAttention(nn.Module):
    """Learned attention over [phase1, phase2, phase3] outputs."""
```

Each node learns a 3-element weight vector (softmax over phases):

```python
stacked = torch.stack([phase1, phase2, phase3], dim=1)  # [N, 3, 256]
scores  = self.attn(stacked)                              # [N, 3, 1]
weights = torch.softmax(scores, dim=1)                    # [N, 3, 1]
output  = (weights * stacked).sum(dim=1)                  # [N, 256]
```

**Phase 1-A2:** Per-phase LayerNorm before JK — prevents Phase 1's higher norm from dominating the softmax.

**Monitored metrics:**
- `last_weights [3]` — mean attention per phase across all nodes
- `last_weight_stds [3]` — std per phase. std ≈ 0 means attention collapsed to constant
- `jk_entropy` — entropy of per-node weights. Low = one phase dominates
- `last_node_weights [N, 3]` — per-node weights in eval mode (for `jk_weight_hist.py` diagnostic)

---

## §4 Edge Masks Forward (lines 500+)

Each phase applies a different edge mask to the same `edge_index`:

```
Phase 1: struct_mask   — edges 0–5 (CALLS..CONTAINS), +self-loops
Phase 2: cfg_mask      — edges 6, 8, 9, 10, 11 (CONTROL_FLOW, ICFG, DEF_USE, EXTERNAL_CALL)
Phase 3: contains_mask — edges 5 forward, 7 reverse (built at runtime)
```

**Fix C1 (line 443):** OOB edge_attr values clamped to `[0, NUM_EDGE_TYPES-1]` before Embedding lookup. Prevents silent CUDA illegal-memory errors from corrupted .pt files.

---

## §5 Run 8 Complexity-Proxy Break (lines 431–437)

```python
if self.drop_complexity:
    x = x.clone()
    x[:, 5] = 0.0
```

Feature index 5 is the complexity feature. Zeroing it forces the model to learn actual vulnerability patterns instead of "big contract = vulnerable." The `.clone()` prevents corrupting the cached graph tensor.

---

## §0.6 Exercises (Q1–Q4)

**Q1: Why 8 layers? Why not 4 or 12?**

2+3+3 matches the graph's natural hierarchy. Phase 1 captures local neighborhood (2 hops = function-internal). Phase 2 captures CFG paths (3 hops = ENTRY→CALL→TMP→WRITE). Phase 3 captures the CONTAINS tree (2 up + 1 down = function→contract→function).

**Q2: What does `add_self_loops=False` in Phase 2/3 achieve?**

Self-loops would let a node attend to itself during CFG message passing. If a node always attends to itself, the directional predecessor→successor signal is diluted. Phase 1 has self-loops because structural aggregation benefits from residual connections. Phase 2/3 removes them because directionality is the signal.

**Q3: The JK attention weights show `last_weights = [0.8, 0.1, 0.1]`. What does this mean?**

Phase 1 dominates (80% weight). The model is bypassing CFG and CONTAINS signals. Possible causes: (a) Phase 1 features are sufficient for the current task, (b) Phase 2/3 have vanishing gradients, (c) JK attention has collapsed. The `jk_entropy` metric confirms — if near 0, the attention is a constant gate. Solution: increase Phase 2/3 learning rate or add aux loss on their outputs.

**Q4: A new edge type `SHADOW_STACK` is added with ID 12. What breaks?**

`NUM_EDGE_TYPES` becomes 12. `self.edge_embedding` needs rebuilding (it was created with the old count). The `cfg_mask` also doesn't include ID 12, so the new edges would only pass through Phase 1. Fix: update `EDGE_TYPES` in `graph_schema.py`, rebuild `edge_embedding`, add to `cfg_mask` if appropriate.
