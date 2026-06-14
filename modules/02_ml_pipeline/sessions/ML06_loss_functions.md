# ML06 — Loss Functions: ASL + Focal Loss

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML06 of 10
**Source:** `ml/src/training/losses.py` (126L), `ml/src/training/focalloss.py` (143L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Multi-label output** (ML01): model produces [B, 10] logits, one per vulnerability class
- **Class imbalance** (Session 31): Reentrancy has 450 positives, most classes have <200
- **Training loop** (ML05): ASL is the default loss function (`loss_fn: "asl"`)

---

## §1 The Imbalance Problem

```
Total cells per batch: B × 10 (batch_size=8 → 80 cells)
Negative cells:        ~85% (most contracts are NonVulnerable for most classes)
```

Standard BCE weights every (class, contract) pair equally. The optimizer spends most gradient budget suppressing easy negatives (DoS=0 on most contracts) rather than lifting rare positives (DoS=1 on 243 train contracts).

---

## §2 Asymmetric Loss (ASL) — Default Loss

Based on Ridnik et al., ICCV 2021: https://arxiv.org/abs/2009.14119

### Key Idea

Apply **different gamma exponents** to positives and negatives, then hard-threshold small negative probabilities:

```python
prob = sigmoid(logits)           # [B, C]

# Clip negatives below threshold to zero
prob_neg = (prob - clip).clamp(min=0.0)  # default clip=0.05

# Focal weights: high weight on HARD examples, low weight on EASY
focal_pos = (1.0 - prob) ** gamma_pos     # gamma_pos=1.0 (mild focus)
focal_neg = prob_neg ** gamma_neg         # gamma_neg=4.0 (aggressive down-weight)

# Loss per cell
loss_pos = -labels * focal_pos * log(prob)
loss_neg = -(1 - labels) * focal_neg * log(1 - prob_neg)
loss = loss_pos + loss_neg
```

### Effect

| Scenario | p | y | BCE weight | ASL weight |
|---|---|---|---|---|
| Easy negative (confident correct) | 0.01 | 0 | 1.0 | ~0 (clipped) |
| Hard negative (model unsure) | 0.40 | 0 | 1.0 | 0.4^4 = 0.026 |
| Easy positive (confident correct) | 0.99 | 1 | 1.0 | 0.01^1 = 0.01 |
| Hard positive (model struggling) | 0.60 | 1 | 1.0 | 0.40^1 = 0.40 |

**Result:** Easy negatives (most of the 410K cells) contribute near-zero gradient. The optimizer focuses on hard negatives and all positives.

### Per-Class ASL (BUG-M3, lines 27–34)

Each of gamma_neg, gamma_pos, clip can be a float (broadcast) or a [C] tensor:

```python
# Softer negative focus for rare classes (e.g., UnusedReturn)
gamma_neg = torch.tensor([4.0, 2.0, 4.0, ...])  # shape [10]
```

Default: all float (uniform across classes).

---

## §3 Focal Loss (lines 8–74) — Alternative

```python
class FocalLoss(nn.Module):
    """Multi-label focal loss with alpha-balancing."""
    # gamma=2.0, alpha=0.25 (paper defaults)
    
    def forward(self, predictions, targets):
        bce = F.binary_cross_entropy(predictions, targets, reduction="none")
        pt = where(targets==1, predictions, 1-predictions)
        alpha_t = where(targets==1, alpha, 1-alpha)
        return (alpha_t * (1-pt)**gamma * bce).mean()
```

**Difference from ASL:**
- Focal: `(1-pt)^gamma` — symmetric on positive/negative
- ASL: separate gamma_pos, gamma_neg — asymmetric focus
- Focal: expects POST-sigmoid probs → needs `_FocalFromLogits` wrapper
- ASL: accepts raw logits directly

**BF16 guard (Audit fix #6, lines 51–58):**
```python
predictions = predictions.float()  # BF16 can't represent 0.001 → 0.0 → log(0) = -inf
```

---

## §4 MultiLabelFocalLoss (lines 77–143) — Per-Class Alpha

```python
class MultiLabelFocalLoss(nn.Module):
    """Per-class alpha, accepts logits (applies sigmoid internally)."""
    
    def __init__(self, alpha: List[float], gamma=2.0):
        # alpha[c] = 1 - n_pos[c] / N  (rare classes get higher alpha)
        self.register_buffer("alpha", torch.tensor(alpha))
```

**Audit fix (2026-05-12):** `alpha_t` is now conditioned on target value:
```python
alpha_t = where(targets==1, alpha, 1.0 - alpha)  # BEFORE: alpha everywhere
```
Old code applied the same alpha to positives AND negatives per class. For rare classes (alpha≈0.9), negative loss was inflated by 0.9 instead of (1-0.9)=0.1 — the model over-penalised correct negatives.

---

## §5 Which Loss When

| Loss | When to use | Default |
|---|---|---|
| **ASL** | Most cases — strong imbalance, multi-label | ✅ **Yes** |
| FocalLoss | Binary tasks, or when weak imbalance | No |
| MultiLabelFocalLoss | Per-class alpha needed, no ASL custom gamma | No |

ASL's default gamma_neg=4.0, clip=0.05 was empirically tuned for SENTINEL's 85% negative rate during Run 7.

---

## §0.6 Exercises (Q1–Q4)

**Q1: Why does ASL with gamma_neg=4.0 work better than BCE for SENTINEL?**

BCE gives equal weight to all 410K (B×C) cells per batch. 85% are negatives where the model is already confident (p < 0.05). ASL's clip=0.05 zeros those out, and gamma_neg=4.0 crushes the remaining easy negatives. The freed gradient budget goes to the 15% hard cells that actually need updating.

**Q2: What happens if clip is set too high (e.g., 0.5)?**

All negatives with p < 0.5 are clipped to zero probability. The model only receives gradient from negatives where it's uncertain (p > 0.5). This would cause the model to stop suppressing negatives once it reaches 50% confidence — too early. clip=0.05 is the empirical sweet spot.

**Q3: The ASL loss becomes NaN mid-training. What's the likely cause?**

BF16 underflow: `prob_neg = (prob - clip).clamp(min=0.0)` → prob_neg could be 0.0 → `log(1 - 0.0) = 0` (fine) or `prob` is 0.0 → `log(0) = -inf`. The `.clamp(min=1e-8)` on `log(prob)` prevents this. But if `prob` is exactly 0.0 in BF16 before the clamp, the damage is done. The explicit `.float()` at line 97 prevents this.

**Q4: TrainConfig has loss_fn="bce" for a quick baseline run. What happens?**

The trainer falls back to `nn.BCEWithLogitsLoss()` (no focal, no ASL). Expected: macro-F1 drops 3-5 points because BCE treats all 85% negatives equally. Useful as a lower-bound comparison.
