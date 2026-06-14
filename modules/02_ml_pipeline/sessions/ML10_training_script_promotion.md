# ML10 — Training Script + Model Promotion

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML10 of 10
**Source:** `ml/scripts/train.py` (350L), `ml/scripts/promote_model.py` (288L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Training loop** (ML05): Trainer handles epochs, aux losses, checkpointing
- **Threshold tuning** (ML09): per-class optimal thresholds saved as JSON
- **MLflow** (Session 21): experiment tracking, artifact registry

---

## §1 train.py (350 LOC) — Training Entry Point

### Naming Convention

```bash
--run-name v10-20260603
```

Always include the date so checkpoints never overwrite each other:

```
ml/checkpoints/v10-20260603_best.pt          ← checkpoint
ml/checkpoints/v10-20260603_best.state.json  ← patience/epoch sidecar
ml/logs/v10-20260603.log                     ← append-mode per-run log
```

### Key CLI Args

| Arg | Default | Note |
|---|---|---|
| `--epochs` | 100 | Full training |
| `--batch-size` | 8 | 8 fits 8GB GPU |
| `--smoke-subsample-fraction` | None | 0.1 = 10% data for 2-epoch smoke test |
| `--no-jk` | (flag) | Ablation: disable JK connections |
| `--resume` | None | Path to checkpoint for resume |
| `--lr` | 2e-4 | Base learning rate |
| `--loss-fn` | `"asl"` | `"bce"`, `"focal"`, or `"asl"` |

### Smoke Test Mode (lines 4-22 example)

```bash
--run-name v7.0-smoke-20260518 --experiment-name sentinel-v7 \
--epochs 2 --smoke-subsample-fraction 0.1
```

Gates that must pass for smoke test:
- GNN gradient share ≥ 15%
- All 3 JK phases have >5% attention weight
- No NaN loss after step 50

### Resume Flow

```python
if args.resume:
    checkpoint = torch.load(args.resume)
    model.load_state_dict(checkpoint["model"])
    optimizer.load_state_dict(checkpoint["optimizer"])
    scheduler = OneCycleLR(optimizer, epochs=config.epochs, ...)  # Fix #32: full epochs
    scheduler.load_state_dict(checkpoint["scheduler"])
    torch.set_rng_state(checkpoint["rng"]["torch"])
```

The resume command is printed at startup so the operator can copy-paste for the next run:

```
INFO: To resume this run:
      TRANSFORMERS_OFFLINE=1 python ml/scripts/train.py \
          --resume ml/checkpoints/v10-20260603_best.pt \
          --run-name v10-20260603-resumed --epochs 100
```

---

## §2 promote_model.py (288 LOC) — MLflow Model Registry

### Why (lines 1–15)

Manual .pt file copying leaves no audit trail and no staged-rollout mechanism. `promote_model.py` logs to MLflow, registers in Model Registry, and transitions stage in one atomic operation.

### Usage

```bash
# Stage to Staging
python ml/scripts/promote_model.py \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --stage Staging \
    --val-f1-macro 0.4812 \
    --note "v2 retrain: edge_attr embeddings (P0-B) active"

# Promote to Production
python ml/scripts/promote_model.py \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --stage Production \
    --val-f1-macro 0.4812 \
    --note "Validated; F1 > 0.4679 gate passed"

# Dry run (no MLflow writes)
python ml/scripts/promote_model.py \
    --checkpoint ml/checkpoints/v10-20260603_best.pt \
    --stage Staging --val-f1-macro 0.4812 --dry-run
```

### What It Does

1. **Load checkpoint metadata** — architecture, num_classes, epoch
2. **Record git commit** — `git rev-parse --short HEAD` for reproducibility
3. **Log to MLflow** — checkpoint as artifact + metadata as params
4. **Register in Model Registry** as `sentinel-vulnerability-detector`
5. **Transition stage** — `Staging` → `Production` (with existing-Production version check)

### Staging Gate Check

```python
# Before promoting to Production, check existing Prod F1
current_f1 = _get_current_production_f1(client)
if args.val_f1_macro <= current_f1:
    log.warning(f"val_f1_macro {args.val_f1_macro} <= current prod {current_f1}")
```

### Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success (or dry-run) |
| 1 | Checkpoint not found, unknown stage, MLflow error |

---

## §3 Full Training-to-Promotion Pipeline

```
1. train.py
   └── checkpoint.pt + state.json + log file + MLflow run

2. calibrate_temperature.py
   └── temperatures.json (per-class temperature scaling)

3. tune_threshold.py
   └── thresholds.json (per-class decision boundaries)

4. Manual validation:
   - Run inference on hold-out test set
   - Check macro-F1, per-class F1, ECE
   - Verify no regression vs. current production model

5. promote_model.py --stage Staging
   └── MLflow Model Registry: Staging

6. Staging validation:
   - Shadow traffic for N hours
   - Compare predictions to production
   - Check drift detector alerts

7. promote_model.py --stage Production
   └── MLflow Model Registry: Production
   └── Inference API auto-loads latest Production model
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: A training run crashes at epoch 37 with a CUDA OOM. Can training be resumed?**

Yes — `train.py --resume ml/checkpoints/v10-20260603_best.pt`. But note: the "best" checkpoint is the one with the highest val F1, which might be at epoch 25 (not 36). The optimizer state and scheduler position are restored. The training resumes from the checkpoint's epoch number. The `state.json` sidecar records the epoch + patience count.

**Q2: promote_model.py reports `val_f1_macro 0.45 <= current prod 0.48`. Does it block promotion?**

It logs a warning but does NOT block — the operator decides. The check exists to prevent accidental regression. If the new model has higher per-class F1 on a specific class (e.g., Timestamp goes from 0.2→0.4 while macro-F1 drops 0.03), the operator may still promote for that specific improvement.

**Q3: What happens if MLflow is offline during promote_model.py?**

MLflow errors → exit code 1. The dry run (`--dry-run`) logs what it WOULD do without contacting MLflow. For offline promotion, manually copy the checkpoint and thresholds to the inference server.

**Q4: The smoke test runs with 10% data for 2 epochs. What gates must it pass?**

Three gates: (1) GNN gradient share ≥ 15% (GNN not collapsed), (2) all 3 JK phases have >5% attention weight (no phase collapsed), (3) no NaN loss after step 50 (no numerical instability). If any gate fails, the smoke test is considered failed and the full run is not started.
