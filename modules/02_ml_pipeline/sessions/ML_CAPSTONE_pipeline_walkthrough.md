# ML_CAPSTONE — End-to-End ML Pipeline Walkthrough

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** Capstone
**Source:** All 10 ML sessions + source code (~5,985 LOC)
**Status:** 🟢 Complete

---

## §1 The Full Flow — One Narrative

### Setup

You're the ML engineer at SENTINEL. The data team just finished a v3 export:
- 22,493 contracts, 21,657 with graph representations
- 10 vulnerability classes (GasException dropped — 0 data)
- Splits: 18,596 train / 1,983 val / 1,914 test — 0% leakage verified

The data module saved:

```
data_module/data/exports/sentinel-v3-smartbugs-2026-06-13/
├── train/
│   ├── graphs_00000.pt    ← 3,720 contracts × 12-dim node features
│   ├── graphs_00001.pt
│   └── ... (5 shards)
├── val/
│   └── graphs_00000.pt    ← 1,983 contracts
├── test/
│   └── graphs_00000.pt    ← 1,914 contracts
└── manifest.json           ← edge types, feature schema, hash
```

### Step 1: Dataset + Collation (ML07)

```python
from ml.src.datasets import SentinelDataset, sentinel_collate_fn

train_dataset = SentinelDataset("data_module/data/exports/.../train", manifest)
train_loader  = DataLoader(train_dataset, batch_size=8, shuffle=True,
                           collate_fn=sentinel_collate_fn,
                           num_workers=4, prefetch_factor=2)
```

The collation function:
1. Pads all graphs in batch to max N nodes
2. Stacks node features → [B, max_N, 12]
3. Creates batch edge_index with offset per graph
4. Pads token sequences to max T tokens
5. Masks padding in attention

### Step 2: Forward Pass Through Four-Eye Model (ML01–ML04)

```python
logits, aux_losses = model(node_feat, edge_index, edge_attr,
                           token_ids, token_mask, batch, cfg_mask)
```

Each batch item passes through:

1. **GNN Encoder** (ML02) — 8-layer GAT with JK, 3 phases:
   - Layer 1–2: structural edges (edges 0–5) → [B, 128]
   - Layer 3–5: +CFG edges (6,8,9,10) → [B, 128]
   - Layer 6–8: +bidir CONTAINS edges → [B, 128]
   - JK aggregates all 8 layers → [B, N, 128]

2. **Transformer Encoder** (ML03):
   - Token IDs → CodeBERT → pool to 128-dim → [B, T, 128]

3. **CrossAttention Fusion** (ML04):
   - GNN→Token attention: each node queries tokens
   - Token→GNN attention: each token queries nodes
   - Pooled → [B, 128]

4. **CFG Eye** — GNN on CFG-only edges → [B, 128]

5. **Concatenation** → [B, 512]

6. **Classifier head** → [B, 10] logits

### Step 3: Training (ML05–ML06, ML10)

```bash
TRANSFORMERS_OFFLINE=1 PYTHONPATH=. python ml/scripts/train.py \
    --run-name v10-20260603 \
    --experiment-name sentinel-v10 \
    --epochs 100 \
    --batch-size 8
```

Training loop:
- Per-group LR: GNN=5e-4, LoRA=6e-5, Fusion=1e-4, Classifier=2e-4
- ASL loss (gamma_neg=4.0, clip=0.05) for main classification
- 3 aux losses (GNN contrastive, token MLM, fusion consistency)
- Warmup: aux losses ramp linearly over 10 epochs
- OneCycleLR: single cycle over all 100 epochs
- Eval every epoch: val F1 macro + per-class
- Checkpoint at best val F1: `ml/checkpoints/v10-20260603_best.pt`
- MLflow log: every metric + config + model artifact

### Step 4: Calibration + Threshold Tuning (ML09)

```bash
python -m ml.scripts.tune_threshold \
    --checkpoint ml/checkpoints/v10-20260603_best.pt

python -m ml.scripts.calibrate_temperature \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --out ml/calibration/temperatures_v10.json
```

Output:
- `v10-20260603_best_thresholds.json` — per-class optimal F1 thresholds
- `temperatures_v10.json` — per-class temperature scalars

### Step 5: Promotion (ML10)

```bash
python ml/scripts/promote_model.py \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --stage Staging \
    --val-f1-macro 0.4812
```

After staging validation (shadow traffic, drift check):

```bash
python ml/scripts/promote_model.py \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --stage Production \
    --val-f1-macro 0.4812
```

### Step 6: Inference (ML08)

```python
from ml.src.inference import Predictor, DriftDetector

predictor = Predictor("ml/checkpoints/v10-20260603_best.pt")
drift     = DriftDetector("ml/checkpoints/v10-20260603_best.pt")

# Inference
result = predictor(smart_contract_graph)
# → {"Reentrancy": 0.93, "Timestamp": 0.18, ...}

# Drift check
alerts = drift.update({"node_count": 42, "edge_count": 89,
                       "reentrancy_prob": 0.93, ...})
```

The API server (FastAPI) serves three-tier suspicion:
- **Confirmed** (≥0.55): immediate on-chain alert
- **Suspicious** (≥0.25): queue for manual review within 24h
- **Noteworthy** (≥0.10): log for weekly report

---

## §2 Key Files Summary

| File | LOC | Primary Role |
|---|---|---|
| `ml/src/models/sentinel_model.py` | 670 | Four-eye orchestration |
| `ml/src/models/gnn_encoder.py` | 667 | 3-phase 8-layer GAT+JK |
| `ml/src/models/transformer_encoder.py` | 388 | CodeBERT + LoRA |
| `ml/src/models/fusion.py` | 295 | Cross-attention fusion |
| `ml/src/training/trainer.py` | 2086 | Training loop, per-group LR, aux losses |
| `ml/src/training/losses.py` | 185 | ASL + Focal loss |
| `ml/src/datasets.py` | 412 | SentinelDataset + collation |
| `ml/src/inference/predictor.py` | 760 | Inference with per-class thresholds |
| `ml/src/inference/api.py` | 402 | FastAPI endpoint |
| `ml/src/inference/drift.py` | 298 | KS-test drift monitor |
| `ml/scripts/train.py` | 350 | Training entry point |
| `ml/scripts/tune_threshold.py` | 616 | Per-class F1 threshold sweeper |
| `ml/scripts/calibrate_temperature.py` | 258 | Per-class temperature scaler |
| `ml/scripts/promote_model.py` | 288 | MLflow Model Registry manager |
| **Total** | **~6,945** | |

---

## §3 Architecture Diagram

```
                    ┌──────────────────────────────────────┐
                    │         Data Module (v3 export)       │
                    │  22,493 contracts → 5 shards/split    │
                    └──────────┬───────────────────────────┘
                               │
                    ┌──────────▼───────────┐
                    │   SentinelDataset    │
                    │   + Collation        │  ← ML07
                    │   LRU cache (4)      │
                    └──────────┬───────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │  GNNEncoder   │   │ Transformer   │   │   CFG Eye    │
  │ 8×GATConv+JK  │   │ CodeBERT+LoRA │   │ 3×GATConv    │
  │ 3-phase mask  │   │ Q+V r=16     │   │ edges 6,8,9,10│
  │ [B,N,128]     │   │ [B,T,128]    │   │ [B,128]       │
  └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
          │                   │                    │
          └───────────┬───────┘                    │
                      ▼                            │
          ┌──────────────────────┐                 │
          │ CrossAttentionFusion │                 │
          │ GNN↔Token attention  │  ← ML04        │
          │ [B,128]             │                 │
          └──────────┬───────────┘                 │
                     │                             │
                     └──────────┬──────────────────┘
                                ▼
                    ┌──────────────────────┐
                    │    Concatenate       │
                    │    [B, 512]          │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   Classifier Head    │
                    │   [B, 10] logits     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   ASL Loss (ML06)    │  ← Training
                    │   or Inference:      │
                    │   ÷ temp → sigmoid   │
                    │   ≥ threshold → pred │
                    └──────────────────────┘
```

---

## §4 Interview Questions

### Mid-Level

**Q: Why does the model use 4 separate eyes instead of one big network?**

Each eye captures a different perspective: GNN sees structural graph patterns, Transformer sees token-level semantics, Fusion sees cross-modal interactions, CFG sees control flow. Concatenating all 4 (128-dim each → 512-dim) prevents any single modality from dominating. If one eye collapses (e.g., GNN oversmoothes), the other 3 still produce useful gradients.

**Q: How does the training loop handle the massive class imbalance?**

Three mechanisms: (1) ASL loss crushes easy negatives (gamma_neg=4, clip=0.05) so 85% of negative cells contribute near-zero gradient; (2) per-class threshold tuning finds the F1-optimal threshold for each class independently; (3) per-group LR compensates for the GNN receiving only ~10% of total gradient (2.5× boost).

**Q: What does JK do and why is it needed?**

GCN/GAT typically use only the last layer's output. In an 8-layer GNN, early layers learn local structure (function boundaries) and late layers learn global structure (cross-contract calls). JK learns a weighted sum of ALL 8 layer outputs, letting the model decide which depth matters per node. Without JK, deeper layers may oversmooth.

**Q: How do you detect dataset drift in production?**

Two-phase: (1) warm-up: collect statistics from first N_warmup requests (feature distributions + label rates); (2) once baseline is set, run KS test on every subsequent batch of 100 predictions against the baseline. If p < 0.01, log WARNING. The drift detector tracks both input features (node count, edge count, complexity) AND output probabilities (per-class mean confidence) for concept drift.

### Senior-Level

**Q: You discover after training that the test_contracts/ benchmark has 20-node median graphs while training had 90-node median. What do you do?**

This is the Run 8 audit finding (OOD problem). Solutions: (1) **Exclude test_contracts/** as benchmark — it's OOD and misleading; switch to `smartbugs-curated/` (median 80 nodes). (2) **Add node-count-based weighting** during training to penalize errors on small contracts more (they're harder). (3) **Synthesize small-contract augmentations** in training (subgraph sampling). (4) **Quantify the gap** in a drift report: feature distribution KS p < 0.01 → "model will perform poorly on this benchmark until retrained on similar-sized contracts."

**Q: The model's F1 dropped from 0.48 to 0.20 on the Monday morning batch. How do you diagnose?**

Check drift detector logs first: did any feature KS p < 0.01 trigger? If node_count distribution shifted (say, Monday batch has 200+ node contracts while training was 90-node median), the GNN may oversmooth. If it's label-rate drift only, the threshold JSON may need recalibration. If it's concept drift (same features, different relationship), the model needs retraining. Also check: was Monday's traffic from a new source not in the training distribution?

**Q: You have training runs consistently plateauing at F1=0.33 (Run 4) while you need F1=0.60. What's your diagnosis plan?**

This is the SENTINEL capacity ceiling. Diagnosis: (1) **Check label quality** — run the 6-stage defense (Phase 5) to estimate noise ceiling (if 30% of positives are wrong, max achievable F1 ≈ 0.60). (2) **Check feature utility** — run interpretability (Layer 4 complexity proxy analysis) to see if the model relies on contract size. (3) **Check split leakage** — re-run leakage auditor; if >0%, retrain on clean splits. (4) **Check GNN gradient share** — if <15%, the GNN is collapsed and needs higher LR or different architecture. (5) **Add more data** — extract BCCC MishandledException, add SmartBugs Curated, add non-vulnerable contracts. Run 12 achieved F1_tuned=0.6941 using exactly these diagnostics.
