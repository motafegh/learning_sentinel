# ML04 — Fusion Layer: Cross-Attention GNN ↔ Tokens

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML04 of 10
**Source:** `ml/src/models/fusion_layer.py` (282L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **GNN encoder** (ML02): outputs node embeddings [N, 256] per contract graph
- **Transformer encoder** (ML03): outputs token embeddings [B, 2048, 768] per contract
- **Four-eye model** (ML01): fused eye uses this layer's output as its 128-dim opinion

---

## §1 What Cross-Attention Solves (lines 1–57)

**Before (concat+MLP):**
```
mean_pool(node_embs) → [B, 256]
mean_pool(token_embs) → [B, 768]
concat([B, 256], [B, 768]) → [B, 1024]
MLP → [B, 64]
```
Problem: Both modalities are pooled to graph-level BEFORE fusion. The `withdraw()` node's structure and the `call.value` token's semantics are averaged with 30 other functions before they can interact.

**After (cross-attention):**
```
Node→Token cross-attention: "withdraw() node finds call.value token"
Token→Node cross-attention: "call.value token finds withdraw() node"
Pool AFTER enrichment: structural patterns find semantic counterparts before averaging
```

Output: [B, 128] (wider than the old [B, 64])

---

## §2 Architecture (lines 26–37)

```
1. Project GNN nodes [N, 256] → [N, 256]     (identity-sized, hidden_dim=256)
2. Project tokens [B, 2048, 768] → [B, 2048, 256]  (common attention dim)
3. Pad nodes → [B, max_nodes, 256] via to_dense_batch(); build node padding mask
4. Node→Token cross-attention (key_padding_mask=token PAD positions)
     → enriched_nodes [B, max_n, 256]
5. Zero-out padded node positions (Fix #8 — structural invariant)
6. Token→Node cross-attention (key_padding_mask=padded node positions)
     → enriched_tokens [B, 2048, 256]
7. Pool enriched nodes — masked mean over real nodes → [B, 256]
8. Pool enriched tokens — masked mean over real tokens → [B, 256]
9. Concat → [B, 512], project → [B, 128]
```

### Dimension details

| Step | Shape |
|---|---|
| Node embeddings | [N, 256] |
| Token embeddings | [B, 2048, 768] |
| After projection | [B, 2048, 256] |
| Dense nodes | [B, max_nodes, 256] |
| Enriched nodes | [B, max_nodes, 256] |
| Enriched tokens | [B, 2048, 256] |
| Pooled nodes | [B, 256] |
| Pooled tokens | [B, 256] |
| Concat → proj | [B, 128] |

---

## §3 Compile-Friendly Scatter (lines 68–156)

```python
def _scatter_to_dense(x, batch, num_graphs, max_nodes):
    """torch.compile-compatible replacement for to_dense_batch."""
```

**Why not `to_dense_batch()`?** PyG's `to_dense_batch()` uses `repeat(size)` which creates a `GuardOnDataDependentSymNode` graph break — forcing the entire forward pass into eager mode. `_scatter_to_dense` uses static `max_nodes` (config constant) for deterministic shapes, keeping the full model `torch.compile()`-compatible.

---

## §4 Fixes Applied (lines 39–57)

| Fix | Issue | Solution |
|---|---|---|
| #2 | Token PAD mask not threaded | Passed to node→token `key_padding_mask` |
| #4 | Device mismatch | Device assertion at forward start |
| #6 | Token pooling included PAD | Masked mean (was plain mean) |
| #7 | Python padding loop → O(N) | `to_dense_batch()` vectorised |
| #8 | Padded nodes had nonzero after attn | Explicit zero-out after enrich |
| #26 | Attention weights allocated but unused | `need_weights=False` saves 12.6MB VRAM |

**Fix #26 detail:** The two attention weight matrices `[B, max_nodes, 2048]` + `[B, 2048, max_nodes]` ≈ 12.6MB per forward. `need_weights=False` lets PyTorch use fused efficient-attention CUDA kernel, saving the allocation and reducing allocator fragmentation.

---

## §5 Training vs. Inference

During training, `CrossAttentionFusion` produces the fused eye (128-dim). During inference, the four eyes are:

```python
cat([gnn_eye(128), transformer_eye(128), fused_eye(128), cfg_eye(128)]) → [B, 512]
```

Cross-attention is the most computationally expensive eye (O(N×T) where N = nodes, T = 2048 tokens). With `need_weights=False` and `torch.compile()`, it's ~2ms per forward on a batch of 32.

---

## §0.6 Exercises (Q1–Q4)

**Q1: A contract has 500 graph nodes and 2048 tokens. What's the attention matrix size?**

Node→Token: [B, max_nodes, 2048]. Token→Node: [B, 2048, max_nodes]. With max_nodes ≈ 557 (99th percentile), each is [B, 557, 2048] ≈ 2.3M floats ≈ 9MB at FP32. With `need_weights=False`, neither is materialized.

**Q2: Why does the fused eye output 128 dims while the old concat+MLP output 64?**

The wider output preserves cross-modal patterns. After cross-attention, each node carries information from specific tokens it attended to — that's richer than a pooled summary, and compressing to 64 would lose too much. 128 balances expressiveness with classifier complexity.

**Q3: What happens if `max_nodes` config is set lower than the actual max nodes in the batch?**

`_scatter_to_dense` would truncate — nodes beyond `max_nodes` are lost. The `_c4_truncation_warned` flag (line 65) logs a warning once. Solution: increase `max_nodes` in `TrainConfig` based on the 99.9th percentile node count from `probe_dataset.py`.

**Q4: How does zeroing padded node positions (Fix #8) affect gradients?**

Padding nodes go through attention and receive nonzero values from token content. Those values are set to zero after enrichment. The pooling step already excludes them via `node_real_mask`, but the nonzero values in the tensor create a gradient path through the attention weights from padding positions. Zeroing eliminates that gradient path — clean gradients from real nodes only.
