# ML03 — Transformer Encoder: CodeBERT + LoRA

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML03 of 10
**Source:** `ml/src/models/transformer_encoder.py` (388L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Tokenizer** (Session 20): GraphCodeBERT tokenizer, 4-window [4, 512], sliding context
- **Four-eye model** (ML01): Transformer eye produces 128-dim from CodeBERT embeddings
- **LoRA**: Low-Rank Adaptation — trainable matrices injected into frozen pretrained weights

---

## §1 Architecture (lines 75–101)

```
CodeBERT (microsoft/graphcodebert-base)
  125M frozen params (never updated)
  + LoRA matrices on query+value of all 12 layers
  ~590K trainable params at r=16
```

**Why not full fine-tune?** 125M params → OOM on 8GB VRAM. Full fine-tune also causes catastrophic forgetting on 41K contracts (CodeBERT was pretrained on 2.3M code-ast pairs).

**Why not fully frozen?** Zero trainable params — CodeBERT never adapts to vulnerability semantics. "transfer" and "call.value" in a vulnerability context differ from "transfer" in an ERC20 context. LoRA adapts Q+V attention to security patterns without touching the 125M frozen weights.

### Input/Output Shapes

| Input dim | Mode | Output dim |
|---|---|---|
| [B, L] | Single-window | [B, L, 768] |
| [B, W, L] | Multi-window (default) | [B, W×L, 768] |

**Multi-window flattening** (lines 255–260): `[B, 4, 512]` → `[B*4, 512]` → CodeBERT → `[B, 2048, 768]`. All 4 windows processed in one fused call, then reassembled.

---

## §2 LoRA Configuration (lines 103–169)

```python
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["query", "value"],
    lora_dropout=0.1,
    bias="none",
    task_type="FEATURE_EXTRACTION",
)
```

**Why Q+V only?** Query and Value projections determine what each token attends to and what information it receives. Key projections are left frozen as the original CodeBERT — the model should attend based on pre-trained semantics, only adapt WHAT it retrieves (Value) and WHAT it looks for (Query).

### LoRA Math

```
Forward pass: W_frozen @ x + (B @ A) @ x × (alpha/r)
              ──────────   ───────────────
              frozen path  trainable LoRA path
```

- A: [768, r] (random init)
- B: [r, 768] (zero init — first forward = frozen only)
- Scale = alpha/r = 32/16 = 2.0
- Gradients flow only through A and B

### Hard Requirement Check (lines 62–72)

```python
if not _PEFT_AVAILABLE:
    raise RuntimeError("peft library is required...")
```

Not a warning, not a fallback — a hard crash. If peft is missing, the model has 0 trainable transformer params and nobody would know until eval. The crash is intentional and non-negotiable.

---

## §3 Forward Pass Details (lines 210–279+)

### Standard path (no prefix)

```python
# Single window
outputs = self.bert(input_ids=input_ids, attention_mask=attention_mask)
return outputs.last_hidden_state  # [B, L, 768]

# Multi-window
flat_ids = input_ids.view(B * W, L)      # [B*4, 512]
flat_mask = attention_mask.view(B * W, L)
outputs = self.bert(input_ids=flat_ids, attention_mask=flat_mask)
return outputs.last_hidden_state.view(B, W * L, 768)  # [B, 2048, 768]
```

### Prefix injection path (lines 262+)

GNN node embeddings are injected as K prefix tokens before the code tokens:

```python
inputs_embeds = torch.cat([gnn_prefix_nodes, word_embs], dim=1)
```

**Purpose:** GNN structure guides CodeBERT attention — "the withdraw() node you just computed should influence how you interpret 'call.value' in the code."

**Position IDs:** Prefix tokens get position_id=1 (RoBERTa's padding position — no positional bias). Code tokens get 3..3+(L-K-1). All within RoBERTa's 514-position limit.

**Gradient note (lines 35–43):** No `torch.no_grad()` around `self.bert()` — peft already marks frozen weights with `requires_grad=False`. PyTorch autograd skips backward graph nodes for frozen ops. Wrapping in `no_grad()` would kill LoRA gradients too.

---

## §4 Flash Attention + Dtype Recovery (lines 128–154)

```python
try:
    self.bert = AutoModel.from_pretrained(
        "microsoft/graphcodebert-base",
        attn_implementation="flash_attention_2",
        torch_dtype=torch.bfloat16,
    )
except (ImportError, ValueError):
    self.bert = AutoModel.from_pretrained(
        "microsoft/graphcodebert-base",
        attn_implementation="sdpa",
    )
finally:
    torch.set_default_dtype(_prev_default_dtype)
```

**Dtype pollution bug:** `from_pretrained` with `torch_dtype=bfloat16` calls `torch.set_default_dtype` as a side effect. Without the `finally` block, every `nn.Linear` created after would default to BF16 — corrupting the GNN encoder and fusion layer weights.

---

## §0.6 Exercises (Q1–Q4)

**Q1: The model trains 10 epochs and the prefix_attn_mean is 0.001. What does this mean?**

The transformer is ignoring the GNN prefix (IMP-M2 gate). Mean attention weight from code tokens to prefix positions is near zero. The prefix nodes are providing no guidance. Possible fix: increase prefix size K, add aux loss on prefix attention, or remove prefix path entirely.

**Q2: Why not LoRA on all 4 attention projections (Q, K, V, O)?**

K determines "what to look for" — should remain CodeBERT's pre-trained distribution. O (output projection) combines head outputs — modifying it would change the fundamental CodeBERT embedding space. Q and V are the minimal adaptation that affects both "what to attend to" (Q) and "what to retrieve" (V).

**Q3: A new PEFT version changes the internal module path. What breaks?**

`_word_embeddings` property (lines 181–208) tries 3 known paths `["base_model.model.embeddings.word_embeddings", "model.embeddings.word_embeddings", "base_model.embeddings.word_embeddings"]`. If none match, it raises AttributeError at construction time — not mid-forward. Fix: add the new path.

**Q4: Flash Attention 2 is not available. What happens?**

The try/except catches ImportError or ValueError (flash-attention not supported), falls back to `attn_implementation="sdpa"` (scaled dot-product attention, PyTorch native). The model works identically — FA2 is a performance optimization, not a correctness requirement.
