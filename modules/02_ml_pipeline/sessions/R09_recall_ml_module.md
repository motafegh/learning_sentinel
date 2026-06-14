# R09 — Recall: ML Sessions 01–10 (ML Pipeline)

**Date:** 2026-06-14
**Session:** R09 of 09
**Purpose:** Final recall — consolidate the entire ML module (architecture → training → promotion)
**Estimated study time:** 25 min
**Status:** 🟢 Complete

---

## §1 Sessions Covered

| # | Session | Topic | Key Files (LOC) |
|---|---|---|---|
| ML01 | Architecture | Four-eye model v8.1 | `sentinel_model.py` (670) |
| ML02 | GNN Encoder | 3-phase 8-layer GAT with JK | `gnn_encoder.py` (667) |
| ML03 | Transformer Encoder | CodeBERT + LoRA (Q+V, r=16) | `transformer_encoder.py` (388) |
| ML04 | Fusion Layer | Cross-attention GNN↔Tokens | `fusion.py` |  |
| ML05 | Training Loop | Per-group LR, OneCycleLR, aux losses | `trainer.py` (2086) |
| ML06 | Loss Functions | ASL (default) + Focal (alternative) | `losses.py` |  |
| ML07 | Dataset + Collation | LRU-cached shard loading | `datasets.py`, `collation.py` |  |
| ML08 | Inference Pipeline | Predictor + API + Drift Detector | `predictor.py` (760), `api.py` (402), `drift.py` |  |
| ML09 | Calibration + Threshold | Per-class T_c + F1-sweep thresholds | `calibrate_temperature.py` (258), `tune_threshold.py` (616) |
| ML10 | Training Script + Promotion | Entry point + MLflow Model Registry | `train.py` (350), `promote_model.py` (288) |

**Total LOC covered:** ~5,985

---

## §2 Key Decisions Recap

### Architecture (ML01)
- **4 independent eyes** (GNN, Transformer, Fused, CFG), each 128-dim → concatenated [B, 512]
- No single modality can dominate; each eye has its own aux loss
- CFG eye is a simple GNN on the control-flow-only edge subset (edges 6, 8, 9, 10)

### GNN Encoder (ML02)
- **3-phase design:** Phase 1 = structural (edges 0–5), Phase 2 = CFG (6, 8, 9, 10), Phase 3 = bidir CONTAINS
- **JK (Jumping Knowledge):** aggregates all 8 layer outputs via learned self-attention weights
- **Edge_attr embeddings:** P0-B learns 128-dim embedding per edge type, concatenated with edge features

### Transformer Encoder (ML03)
- **CodeBERT frozen (125M params), LoRA adapters on Q+V only (r=16)**
- LoRA rank 16 because 8 was unstable in ablation; 32 added no gain
- `peft` NOT in root venv — hard RuntimeError if missing

### Fusion Layer (ML04)
- **Cross-attention replaces concat+MLP**
- GNN→Token attention + Token→GNN attention before pooling
- Lets `withdraw()` node attend to `call.value` token at fine granularity

### Training Loop (ML05)
- **Per-group LR:** GNN 2.5× base, LoRA 0.3×, Fusion 0.5×, Classifier 1.0×
- **Aux losses warmup:** linear ramp over 10 epochs
- **OneCycleLR:** single cycle over ALL epochs (Fix #32)
- **ASL default** (ML06): gamma_neg=4, clip=0.05 — crushes 85% easy negatives

### Dataset (ML07)
- **LRU cache of 4 shards** — configurable via `SENTINEL_SHARD_CACHE_SIZE`
- **SentinelCollation** handles variable-length graphs + token windows
- **Fix #28:** PyG v2.6 `edge_index` must be LongTensor, not int

### Inference (ML08)
- **3-tier suspicion:** confirmed (≥0.55), suspicious (≥0.25), noteworthy (≥0.10)
- **Drift detection:** two-phase warmup → KS baseline → continuous alerts
- **Predictor warmup** uses 2-node/1-edge graph (1-edge skips GATConv.propagate())

### Calibration + Threshold (ML09)
- **Per-class temperature T_c:** fits one scalar per class by minimising BCE NLL
- **Per-class threshold:** sweeps 0.01–0.99 by 0.01 to maximise class-level F1
- Minority classes (UnusedReturn) get thresholds as low as 0.12

### Promotion (ML10)
- **MLflow Model Registry** with Staging → Production stages
- **Dry-run mode** — no MLflow writes, prints planned actions
- **Gate check:** warns if new val_f1_macro ≤ current production value

---

## §3 Pipeline — From Raw Graph to Production Deployment

```
graph.pt + tokens.pt                        ← Data module v3 export
       │
       ▼
SentinelDataset + SentinelCollation        ← ML07: LRU-cached shards
       │
       ▼
train.py                                    ← ML10: entry point
  │
  ├── GNNEncoder (3-phase 8-layer GAT+JK)   ← ML02
  ├── TransformerEncoder (CodeBERT+LoRA)    ← ML03
  ├── CrossAttentionFusion                  ← ML04
  ├── ClassifierHead → [B, 10] logits       ← ML01
  └── Trainer (per-group LR, ASL, aux)       ← ML05+ML06
       │
       ▼
checkpoint.pt
       │
       ├──→ calibrate_temperature.py        ← ML09: per-class T_c
       ├──→ tune_threshold.py               ← ML09: per-class thresholds
       └──→ promote_model.py                ← ML10: MLflow Staging
              │
              ▼
Inference: predictor.py → api.py → Drift Detector  ← ML08
```

---

## §4 Quick-Reference Tables

### Model Dimensions

| Component | Input | Hidden | Output |
|---|---|---|---|
| GNN encoder | [B, N, 12] | 128 | [B, N, 128] |
| Transformer encoder | [B, T] (token IDs) | 768 → 128 | [B, T, 128] |
| Fusion (pooled) | [B, N, 128] + [B, T, 128] | 128 | [B, 128] |
| CFG eye | [B, N_CFG, 12] | 128 | [B, 128] |
| Concatenated | — | — | [B, 512] |

### Hyperparameters

| Param | Value |
|---|---|
| LoRA rank | 16 |
| GNN layers | 8 |
| JK attention heads | 3 (one per phase) |
| Temperature scaling | Per-class |
| Default threshold | 0.5 (fallback) |
| Threshold sweep step | 0.01 |
| LRU cache size | 4 shards |
| Aux loss warmup | 10 epochs |
| ASL gamma_neg | 4.0 |
| ASL clip | 0.05 |
| Focal gamma | 2.0 |
| Focal alpha | 0.25 |

### Threshold Tiers

| Tier | Threshold | Meaning |
|---|---|---|
| Confirmed | ≥ 0.55 | Strong signal, high precision |
| Suspicious | ≥ 0.25 | Moderate signal, monitor |
| Noteworthy | ≥ 0.10 | Weak signal, log for later review |

---

## §5 Exercise Questions

**Q1: What is the purpose of JK (Jumping Knowledge) in the GNN encoder?**

It aggregates all 8 GNN layer outputs via learned self-attention weights (one weight per phase: structural/CFG/CONTAINS). Without JK, deeper layers may oversmooth and lose phase-1 structural information. With JK, the model can attend to early layers for structural patterns and late layers for long-range CFG dependencies.

**Q2: Why does the GNN need a 3-phase design instead of a single forward pass with all edges?**

Because different edge types encode fundamentally different relationships. Structural edges (0–5) are syntactic — function calls, inheritance, state reads. CFG edges (6, 8, 9, 10) are sequential — which instruction follows which. CONTAINS edges are hierarchical — which function is in which contract. Mixing them in a single phase dilutes each signal. Phase masking lets the model learn distinct representations for each relationship type.

**Q3: The predictor returns {"Reentrancy": 0.93, "Timestamp": 0.18} but thresholds.json has Timestamp at 0.12 and Reentrancy at 0.55. What is the suspicion tier for each?**

Reentrancy: 0.93 ≥ 0.55 → **Confirmed**. Timestamp: 0.18 ≥ 0.12 but < 0.25 → **Noteworthy** (not suspicious). The Timestamp prediction, though above its optimal threshold, is so low that it only triggers the lowest tier.

**Q4: Train with --smoke-subsample-fraction 0.1 --epochs 2. What 3 gates must pass?**

(1) GNN gradient share ≥ 15% (GNN not collapsed into a constant), (2) all 3 JK phases have >5% attention weight (no dead phase), (3) no NaN loss after step 50 (no numerical instability, gradient explosion, or division by zero).
