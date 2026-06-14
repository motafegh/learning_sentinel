# ML01 — SENTINEL Four-Eye Model Architecture (v8.1)

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML01 of 10
**Source:** `ml/src/models/sentinel_model.py` (670L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Export shards** (Session 28–29): graphs.pt + tokens.pt split by contract_id
- **Graph schema** (Session 15): 11 node features, 11 edge types, 13 node types
- **Tokenizer** (Session 20): 4-window, 512 tokens each, CodeBERT-compatible
- **10-class output** (Session 22): 9 vulnerability classes + NonVulnerable

---

## §1 The Four-Eye Principle

The model uses **4 independent eyes** that each produce a 128-dim opinion vector, concatenated to [B, 512], then classified. Each eye sees the contract through a different structural lens:

```
                     ┌──────────────┐
  GNN eye (128) ─────┤              │
  Transformer eye    │  cat → [512] │
    (128)       ─────┤ → Linear(256)│
  Fused eye          │ → ReLU/Drop  │
    (128)       ─────┤ → Linear(10) │
  CFG eye (128) ─────┤              │
                     └──────────────┘
```

This prevents any single modality from dominating. Each eye must contribute to solve the 10-class task.

---

## §2 The Four Eyes

### GNN Eye (lines 15–17)
```
Pool over FUNCTION/MODIFIER/FALLBACK/RECEIVE/CONSTRUCTOR nodes
global_max_pool + global_mean_pool → [B, 512] → gnn_eye_proj → [B, 128]
```

Structural opinion from Phase 3 GNN output. Only function-level nodes pooled — the full contract structure condensed.

### Transformer Eye (lines 19–20)
```
WindowAttentionPooler → [B, 768] → transformer_eye_proj → [B, 128]
```

Semantic opinion from CodeBERT token embeddings. Four 512-token windows pooled into one representation.

### Fused Eye (lines 22–23)
```
CrossAttentionFusion(node_embs, token_embs) → [B, 128]
```

Joint structural+semantic opinion. GNN node embeddings attend to token embeddings and vice versa before pooling. "`withdraw()` node attends to `call.value` token."

### CFG Eye (lines 25–28)
```
Pool _phase2_x over CFG_NODE types [8-12] — nodes receiving Phase 2 messages
global_max_pool + global_mean_pool → [B, 512] → cfg_eye_proj → [B, 128]
```

Direct gradient path from conv3/conv3b/conv3c (BUG-R7-1 fix). Phase 2 is the CFG-aware layers — this eye captures intra-function execution ordering without going through Phase 3's CONTAINS pass.

---

## §3 Auxiliary Heads (lines 34–38)

Training only — zero inference overhead:

```python
aux_gnn         = Linear(128, num_classes)(gnn_eye)
aux_transformer = Linear(128, num_classes)(transformer_eye)
aux_fused       = Linear(128, num_classes)(fused_eye)
aux_phase2      = MLP(256→128→num_classes) pooled over CFG nodes
```

**Purpose:** Prevents eye dominance. If the GNN eye dominates, the transformer aux head's loss will be high, forcing the model to learn from all modalities. Each aux head sees `num_classes` (10) output. Total aux loss = sum of all 4 aux BCE losses.

**Return signature:**
```python
forward(x, return_aux=True) → (logits, {"gnn", "transformer", "fused", "phase2", "jk_entropy"})
forward(x, return_aux=False) → logits only
```

---

## §4 Type Embedding Assert (lines 71–78)

```python
_MAX_TYPE_ID = float(max(NODE_TYPES.values()))  # 13.0 for v9 schema
assert _MAX_TYPE_ID == 13.0, (
    f"_MAX_TYPE_ID is {_MAX_TYPE_ID} but expected 13.0 (v9 schema). "
    "A new node type was added — update normalization in graph_extractor.py "
    "and sentinel_model.py before re-extracting graphs."
)
```

**Design invariant:** The graph extractor stores node type as `float(type_id) / 13.0`. The model recovers it by multiplying by 13.0 and rounding. If a new node type is added (making max 14.0), this assert fires at import time — catching misalignment BEFORE silent corruption.

---

## §5 Forward Pass Flow

```
Input: Batch (graph batch) + token_ids [B, 4, 512] + attention_mask [B, 4, 512]

1. GNNEncoder(batch) → node_embs [N, 256], phase2_x [N, 256], phase_outputs [3, N, 256]
   └── JK attention → jk_entropy [B, num_layers]

2. TransformerEncoder(token_ids, attention_mask) → token_embs [B, 512, 768]

3. GNN Eye:   pool(function nodes).max + .mean → concat → [B, 512] → proj → [B, 128]
   CFG Eye:   pool(CFG nodes from phase2_x).max + .mean → [B, 512] → proj → [B, 128]

4. Transformer Eye: WindowAttentionPooler(token_embs) → [B, 768] → proj → [B, 128]

5. Fused Eye: CrossAttentionFusion(node_embs, token_embs) → [B, 128]

6. cat([gnn, transformer, fused, cfg]) → [B, 512]
   → Linear(512, 256) → ReLU → Dropout → Linear(256, 10) → [B, 10]
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: Why four eyes instead of one big model?**

Each eye captures a different inductive bias: GNN for structure, Transformer for semantics, Fused for cross-modal patterns, CFG for intra-function control flow. If one modality is noisy (e.g., CodeBERT misclassifies a token), the other three eyes can compensate. The auxiliary heads ensure no single eye dominates.

**Q2: What does the CFG eye add that the GNN eye doesn't?**

The GNN eye pools Phase 3 (post-CONTAINS propagation). The CFG eye pools Phase 2 (direct GATConv output from CFG-specific layers). This gives gradient direct access to conv3/conv3b/conv3c without going through Phase 3's bidirectional CONTAINS pass — which could smear CFG-specific signals across function boundaries.

**Q3: How is the type embedding assert triggered?**

If someone adds a new node type to `graph_schema.NODE_TYPES` (e.g., `"YUL_BLOCK": 14`), `_MAX_TYPE_ID` becomes 14.0. The assert `_MAX_TYPE_ID == 13.0` fires at module import. Fix: (1) update normalization divisor in `graph_extractor.py`, (2) update `_MAX_TYPE_ID` here, (3) re-extract graphs.

**Q4: Why `return_aux=False` during inference?**

Aux heads are Linear layers that would be computed but their output is unused during inference. `return_aux=False` skips them entirely — zero extra computation, zero VRAM. The JK entropy is also skipped (used only for monitoring training dynamics).
