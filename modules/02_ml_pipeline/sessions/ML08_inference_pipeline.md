# ML08 — Inference Pipeline: Predictor + API + Drift + Preprocess

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML08 of 10
**Source:** `predictor.py` (760L), `api.py` (402L), `preprocess.py` (625L), `drift_detector.py` (193L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Trained model** (ML05): checkpoint with model weights + config saved
- **Export artifact** (ML07): sharded .pt files consumed at training time
- **Inference vs. training**: No aux heads, no gradient, single-contract input

---

## §1 ContractPreprocessor (625 LOC) — Source → Model Input

### Entry Points (lines 19–30)

```python
process(sol_path)              # contract already on disk (CLI)
process_source(source_code)    # raw string from HTTP API
process_source_windowed(source_code)  # sliding-window for long contracts
```

### What It Does

1. **Slither temp-file** (for `process_source`): `NamedTemporaryFile` → solc requires a real path, cannot read from a pipe
2. **Graph extraction**: calls `extract_contract_graph()` from `ml/src/preprocessing/graph_extractor.py`
3. **Tokenization**: CodeBERT tokenizer, 512-token windows
4. **Hashing**: SHA-256 via `hash_utils.py` (replaces old inline md5)
5. **Exception translation**: `GraphExtractionError` → `ValueError` (HTTP 400) or `RuntimeError` (HTTP 500)

### Thin Wrapper Design

Graph construction lives in `ml/src/preprocessing/graph_extractor.py`. This file ONLY handles:
- Temp file management
- Exception translation
- Tokenization
- Hashing
- Response logging

---

## §2 Predictor (760 LOC) — Model Loader + Scorer

### Checkpoint Loading (lines 1–60 of header)

```python
class Predictor:
    def __init__(self, checkpoint_path, device="cuda", threshold=0.5):
        # 1. Load checkpoint config
        saved_cfg = checkpoint.get("config", {})
        arch = saved_cfg.get("architecture", "unknown")
        
        # 2. Determine fusion_output_dim
        fusion_dim = saved_cfg.get("fusion_output_dim")
        if fusion_dim is None:
            fusion_dim = _ARCH_TO_FUSION_DIM.get(arch, 128)
        
        # 3. Build model
        model = SentinelModel(num_classes=NUM_CLASSES, fusion_output_dim=fusion_dim, ...)
        model.load_state_dict(checkpoint["model"])
        
        # 4. Per-class thresholds
        self.thresholds = self._load_thresholds(checkpoint_path)  # from {checkpoint.stem}_thresholds.json
```

**Architecture compatibility** (Bug 4): Unknown architecture → `ValueError`. An allowlist prevents silent fallthrough to wrong fusion dim.

**Per-class thresholds** (Fix #6): Returns `"thresholds": [0.3, 0.5, 0.4, ...]` per class instead of a single global threshold. Tuned by `tune_threshold.py`.

### Warmup (Audit #5, Fix #4)

```python
def _warmup(self):
    dummy_x = torch.randn(2, NODE_FEATURE_DIM)    # 2 nodes
    edge_index = torch.tensor([[0], [1]])           # 1 edge
    edge_attr = torch.tensor([0])                   # structural edge type
    dummy_tokens = {"input_ids": ..., "attention_mask": ...}
    self.model(dummy_x, edge_index, ..., dummy_tokens)
```

**Why a 2-node 1-edge graph?** A 0-edge graph skips GATConv.propagate() entirely — hiding shape bugs until first real request. 1 edge exercises the full message-passing path.

### Score (__call__)

```python
def __call__(self, source_code):
    graph, tokens = self.preprocessor.process_source(source_code)
    model_input = {graph.x, graph.edge_index, ..., tokens}
    logits = self.model(**model_input, return_aux=False)
    probs = torch.sigmoid(logits)  # [10]
    
    predictions = {}
    for i, class_name in enumerate(CLASS_NAMES):
        if probs[i] >= self.thresholds[i]:
            predictions[class_name] = probs[i].item()
    
    return {
        "probabilities": dict(zip(CLASS_NAMES, probs.cpu().tolist())),
        "predictions": predictions,
        "thresholds": self.thresholds.cpu().tolist(),
    }
```

---

## §3 API (402 LOC) — FastAPI Endpoint

### Three-Tier Suspicion (2026-05-27 schema)

```python
class PredictResponse(BaseModel):
    label: str                      # "safe" | "suspicious" | "confirmed_vulnerable"
    probabilities: dict[str, float]  # full 10-class vector
    confirmed: list[dict]           # probability >= 0.55
    suspicious: list[dict]          # probability >= 0.25
    tier_thresholds: dict           # {"confirmed": 0.55, "suspicious": 0.25, "noteworthy": 0.10}
```

### Request Flow

```
POST /predict
    → preprocess (CompilePreprocessor)
    → predictor (SentinelModel forward)
    → drift_detector.update_stats(...)  # every request
    → every 50th request: drift_detector.check()
    → PredictResponse
```

### Prometheus Monitoring

```python
_gauge_model_loaded  = Gauge("sentinel_model_loaded", ...)
_gauge_gpu_mem_bytes = Gauge("sentinel_gpu_memory_bytes", ...)
_gpu_oom_counter     = Counter("sentinel_gpu_oom_total", ...)
```

**OOM recovery:** On `torch.cuda.OutOfMemoryError`, the API runs `torch.cuda.empty_cache()` and returns HTTP 503.

---

## §4 DriftDetector (193 LOC) — Production Input Monitoring

### Why (lines 1–15)

SENTINEL was trained on BCCC-SCsVul-2024 (2024 snapshot). Production contracts from 2026+ may have different distributions. KS test detects silent accuracy degradation.

### Two-Phase Baseline Strategy

```
Phase 1 (warm-up): Collect stats from first N_WARMUP requests. Suppress alerts.
Phase 2 (active):  Write drift_baseline.json. Enable KS alerts.
```

### Per-Request + Periodic Check

```python
detector.update_stats({"num_nodes": n, "num_edges": e, ...})  # every request

if request_count % DRIFT_CHECK_INTERVAL == 0:
    detector.check()  # KS test: p < 0.05 → Prometheus alert
```

### Features Checked

- `num_nodes`, `num_edges` — graph size
- `cyclomatic_complexity`, `call_depth` — CFG structure
- `function_count`, `loc` — contract scale

---

## §0.6 Exercises (Q1–Q4)

**Q1: The API returns label="safe" but probabilities show Reentrancy=0.52. How is this possible?**

The three-tier thresholds: confirmed=0.55, suspicious=0.25, noteworthy=0.10. 0.52 is above suspicious (0.25) but below confirmed (0.55). It would appear in `suspicious` list but the overall `label` is "safe" because no class exceeds the "confirmed" threshold. This is correct behavior — 0.52 is not confident enough for "confirmed_vulnerable."

**Q2: The drift detector fires an alert on num_nodes. What should the operator do?**

Check if the production contracts have genuinely changed (new Solidity version, different contract patterns) or if the baseline was computed incorrectly. Phase 1 warm-up before alerting prevents false positives from the initial request distribution.

**Q3: Why does `process_source` write a temp file for Slither?**

Solc requires a real file path — it cannot read from a pipe or stdin argument. The temp file is created in a `NamedTemporaryFile` with `delete=False`, passed to Slither's `Slither(path)`, and cleaned up in a `finally` block.

**Q4: The predictor loads a checkpoint saved with fusion_output_dim=128 but the new model defaults to 64. What happens?**

The checkpoint config stores `fusion_output_dim: 128`. The predictor reads it via `saved_cfg.get("fusion_output_dim")` → uses 128. If the key is missing (legacy checkpoint), it falls back to `_ARCH_TO_FUSION_DIM[arch]`. The model builds with the correct dim either way.
