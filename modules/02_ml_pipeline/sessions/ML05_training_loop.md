# ML05 — Training Loop (v8.1)

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML05 of 10
**Source:** `ml/src/training/trainer.py` (2,086L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Four-eye model** (ML01): GNN + Transformer + Fused + CFG, each with 128-dim aux head
- **Loss functions** (covered in ML06): AsymmetricLoss, FocalLoss
- **Dataset** (ML07): SentinelDataset loads export shards by contract_id

---

## §1 TrainConfig (lines 186–300+) — The Config Hub

Key defaults:

| Param | Value | Why |
|---|---|---|
| `epochs` | 100 | v6 was 60; more data + harder ASL loss needs more epochs |
| `batch_size` | 8 | Fits 8GB GPU with MAX_WINDOWS=4 (16 saturates 7.9/8.0 GB) |
| `lr` | 2e-4 | Base LR, modified by per-group multipliers |
| `gnn_lr_multiplier` | 2.5 | GNN gets 2.5× base LR — it collapsed to ~10% gradient share |
| `lora_lr_multiplier` | 0.3 | LoRA runs at 0.3× base LR — prevents CodeBERT forgetting |
| `fusion_lr_multiplier` | 0.5 | Fusion at 0.5× — its norms were 4-5× higher than GNN |
| `fusion_max_nodes` | 2048 | 99.9th percentile; BUG-C4: 227 contracts exceed 1024 |
| `loss_fn` | `"asl"` | Asymmetric Loss (ICCV 2021) for multi-label imbalance |
| `gradient_accumulation_steps` | 1 | Effective batch = batch_size × N |

### Per-Group LR (Phase 2-B1, lines 232–244)

```python
optimizer = AdamW([
    {"params": gnn_params,  "lr": lr * 2.5},   # GNN: boost
    {"params": lora_params, "lr": lr * 0.3},   # LoRA: careful
    {"params": fusion_params, "lr": lr * 0.5}, # Fusion: moderate
    {"params": classifier_params, "lr": lr},   # Classifier: base
])
```

**Why split?** The GNN collapsed to ~10% of total gradient share by epoch 8 in v5.1. LoRA at full LR would catastrophically forget CodeBERT features that took many epochs to adapt. Fusion at full LR produced 4-5× higher gradient norms than GNN (RC1 fix).

---

## §2 Aux Loss Warmup (Fix #33, lines 40–44)

```python
# aux_loss_weight ramps linearly from 0 → configured value over N epochs
if epoch < aux_loss_warmup_epochs:
    weight = aux_loss_weight * (epoch / aux_loss_warmup_epochs)
```

**Problem:** The 4 auxiliary heads each produce a BCE loss on 10 classes. Early in training, the main classifier hasn't learned anything, but the aux heads produce losses 2-4× higher. Without warmup, aux losses dominate gradients before the main classifier can establish useful representations.

**Solution:** Linear ramp from 0 to full weight over `aux_loss_warmup_epochs` (default 10).

### Total Loss (each step)

```
loss = main_asl_loss
     + aux_loss_weight * (aux_gnn_loss + aux_transformer_loss + aux_fused_loss + aux_phase2_loss)
     + jk_entropy_weight * jk_entropy
```

---

## §3 Scheduler: OneCycleLR (lines 35–39)

```python
scheduler = OneCycleLR(
    optimizer,
    max_lr=lr,
    epochs=epochs,           # Fix #32: full epoch count, NOT remaining_epochs
    steps_per_epoch=steps,
)
```

**Fix #32 — Resume bug:** OneCycleLR was created with `epochs=remaining_epochs` on resume. The scheduler's `total_steps` = `epochs × steps_per_epoch` would be wrong by the number of completed epochs, causing `state_dict.load_state_dict()` to silently discard the checkpoint scheduler state. Fix: always create with the full epoch count.

---

## §4 Training Step Flow (simplified)

```
for epoch in range(start_epoch, config.epochs):
    model.train()
    for batch in train_loader:
        # 1. Forward
        logits, aux = model(batch, return_aux=True)
        
        # 2. Compute losses
        main_loss = asl_loss(logits, batch.labels)
        aux_loss = sum(aux_heads) * warmup_weight
        total_loss = main_loss + aux_loss + jk_entropy * jk_weight
        
        # 3. Backward (AMP scaler)
        scaler.scale(total_loss).backward()
        
        # 4. Gradient accumulation
        if (step + 1) % accumulation_steps == 0:
            scaler.unscale_(optimizer)
            grad_norm = torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm)
            scaler.step(optimizer)
            scaler.update()
            optimizer.zero_grad(set_to_none=True)   # Fix #28
            scheduler.step()
    
    # 5. Eval on val set
    val_logits = model(val_batch, return_aux=False)
    val_f1 = f1_score(val_labels, val_logits > threshold, average="macro")
    
    # 6. Checkpoint if best F1
    if val_f1 > best_f1:
        save_checkpoint({
            "model": model.state_dict(),
            "optimizer": optimizer.state_dict(),
            "scheduler": scheduler.state_dict(),
            "rng": torch.get_rng_state(),
            "best_f1": val_f1,
        })
    
    # 7. GC + empty_cache between epochs (Fix #27)
    gc.collect()
    torch.cuda.empty_cache()
```

---

## §5 Speed Optimizations (lines 5–17)

1. **AMP** — Automatic Mixed Precision (BF16/FP16)
2. **TF32** — `torch.set_float32_matmul_precision("high")`
3. **num_workers=2** — parallel data loading
4. **pin_memory=True** — faster CPU→GPU transfer
5. **zero_grad(set_to_none=True)** — frees gradients immediately (vs. zeroing)
6. **Gradient accumulation** — effective batch = batch_size × N
7. **gc.collect() + empty_cache() between epochs** (Fix #27)

---

## §6 Safe Resume (Fix #35, lines 46–52)

```python
checkpoint = torch.load(path)
model.load_state_dict(checkpoint["model"])
optimizer.load_state_dict(checkpoint["optimizer"])
scheduler.load_state_dict(checkpoint["scheduler"])
torch.set_rng_state(checkpoint["rng"]["torch"])
```

**Fix #35:** Saves RNG states (torch/CUDA/numpy/python) and per-class threshold cache in every best-checkpoint. Default `resume_model_only=False` — full resume always. Without this, Adam's momentum/variance were discarded on resume, causing a 5-10 epoch cold-start regression.

---

## §0.6 Exercises (Q1–Q4)

**Q1: The training log shows GNN gradient norm = 0.02 and LoRA norm = 0.85 for 10 epochs. What's happening?**

GNN is collapsed — its gradients are 40× smaller than LoRA's. The per-group LR multiplier should fix this (GNN gets 2.5×, LoRA gets 0.3×), but if the collapse is structural (not just LR), check: (a) JK attention weights — is Phase 1 dominating at 80%+? (b) Are Phase 2/3 edge masks empty? (c) Is the GraphCodeBERT signal dominating via CrossAttentionFusion?

**Q2: The loss spikes to NaN at epoch 3. Where do you look first?**

Three common causes: (1) BF16 underflow in FocalLoss — `prob=0.0` → `log(0)=-inf` (Fix #6's `.float()` cast catches this). (2) OneCycleLR at peak LR — reduce `max_lr` or extend warmup. (3) Corrupted graph .pt with NaN features — validate with `validate_graph_integrity=True`.

**Q3: Why is aux_loss_warmup=10 epochs? Doesn't that delay learning?**

The main classifier loss (ASL) trains from epoch 0 — only the aux heads are warmupped. The main loss dominates early epochs anyway (it sees 10 classes × batch_size samples per step). The warmup prevents aux heads from adding 2-4× extra loss before the main classifier has any useful signal.

**Q4: Training stops at epoch 37 and the user uses --resume. What state is restored?**

Full state from the best checkpoint: model weights, AdamW momentum+variance, OneCycleLR position, RNG states (torch/CUDA/numpy/python), per-class threshold cache. Training starts from the checkpoint's epoch number, not 0. OneCycleLR uses the full epoch count (Fix #32) so the scheduler position loads correctly.
