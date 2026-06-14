# ML07 — Dataset + Collation: Loading Export Shards

**Date:** 2026-06-14
**Module:** 02_ml_pipeline
**Session:** ML07 of 10
**Source:** `ml/src/datasets/sentinel_dataset.py` (193L), `ml/src/datasets/collate.py` (51L)
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Export artifact** (Session 28): `labels.parquet`, `graphs-{shard}.pt`, `tokens-{shard}.pt`
- **Shard index** (Session 28): `{sha256: {shard, pos_in_shard, num_nodes}}`
- **Four-eye model** (ML01): expects PyG Data + tokens dict per contract

---

## §1 SentinelDataset (193 LOC) — PyTorch Dataset

### Construction (lines 57–158)

Three hard gates at init, all raise `ValueError` on failure:

1. **Format schema version** — must be `"v1"`. Catches stale export artifacts.
2. **Graph schema version** — must match `FEATURE_SCHEMA_VERSION`. Prevents silent feature drift when `graph_schema.py` changes.
3. **Artifact hash integrity** — verifies SHA-256 over all data files. Catches corruption or tampering.

### Label Lookup (lines 112–119)

```python
labels_df = pd.read_parquet(self.export.labels_path).set_index("contract_id")
for cid, row in labels_df.iterrows():
    y = torch.tensor([row[f"class_{i}"] for i in range(10)], dtype=torch.float32)
    tier = None if pd.isna(row["confidence_tier"]) else str(row["confidence_tier"])
    self._label_lookup[cid] = (y, tier)
```

Reads once at init, stores as `{sha256: (y_tensor, tier)}` — O(1) lookup per `__getitem__`.

### Contract ID List (lines 123–127)

```python
all_ids = self.export.get_split_contract_ids(split)
self._contract_ids = [sha for sha in all_ids if sha in shard_index]
```

Filters to contracts that HAVE graph representations. Contracts with labels but no .pt are excluded (they're in `labels.parquet` but not in shards).

### Per-Contract num_nodes (lines 129–157)

Two paths:
- **Fast path**: `shard_index[sha]["num_nodes"]` — read from manifest (new exports)
- **Slow fallback**: Load each graph shard, call `get_example(pos).num_nodes`. Sorted by (shard, pos) so each shard loads once and stays hot in LRU cache.

Used by the trainer's timestamp-size weighted sampler.

### __getitem__ (lines 172–193)

```python
def __getitem__(self, idx):
    contract_id = self._contract_ids[idx]
    entry = self.export.shard_index[contract_id]
    
    graph_shard = _load_graph_shard(f"graphs-{entry['shard']:05d}.pt")
    graph = graph_shard.get_example(entry['pos_in_shard'])
    
    token_shard = _load_token_shard(f"tokens-{entry['shard']:05d}.pt")
    input_ids = token_shard[entry['pos_in_shard']]  # [4, 512]
    attention_mask = (input_ids != _PAD_TOKEN_ID).long()
    
    y, tier = self._label_lookup[contract_id]
    return graph, {"input_ids": input_ids, "attention_mask": attention_mask}, y, contract_id, tier
```

### Shard Caching (lines 47–54)

```python
@lru_cache(maxsize=4)
def _load_graph_shard(path): return torch.load(path, weights_only=False)

@lru_cache(maxsize=4)
def _load_token_shard(path): return torch.load(path, weights_only=True)
```

LRU cache avoids reloading the same shard for every contract. `maxsize=4` means 4 graph + 4 token shards in memory simultaneously. With shard_size=5000 and batch_size=8, a batch of 8 contracts from the same shard → 1 cache hit. A batch of 8 across 2 shards → 2 loads.

---

## §2 Collation (`sentinel_collate_fn`, 51 LOC)

### Input → Output

```
Input:  list of (graph, tokens_dict, y, contract_id, tier) — one per item
Output: (graphs_batch, tokens_batch, y_batch, contract_ids, tiers)
```

### PyG Batch (lines 41–43)

```python
graphs_batch = Batch.from_data_list(graphs, exclude_keys=_EXCLUDE_KEYS)
```

`_EXCLUDE_KEYS` includes `contract_hash`, `contract_path`, `num_nodes`, `y` — string/non-tensor metadata that `Batch.from_data_list()` can't merge.

### Token Stacking (lines 45–47)

```python
input_ids = torch.stack([t["input_ids"] for t in tokens_list])        # [B, 4, 512]
attention_mask = torch.stack([t["attention_mask"] for t in tokens_list])  # [B, 4, 512]
```

Simple stack — all contracts have the same token shape [4, 512].

### Label Stacking (line 49)

```python
y_batch = torch.stack(list(ys))  # [B, 10]
```

Straightforward — all labels are [10]-dim float tensors.

---

## §3 End-to-End Flow

```
SentinelDataset.__getitem__(idx)
    → shard_index → graph shard + position
    → load graph shard (LRU cache) → get_example(pos) → PyG Data
    → load token shard (LRU cache) → slice [pos] → [4, 512] → build mask
    → label_lookup → y [10], tier
    → returns (Data, tokens_dict, y, sha256, tier)

sentinel_collate_fn([items...])
    → Batch.from_data_list(graphs)
    → stack tokens
    → stack ys
    → returns (Batch, tokens_dict, y_batch, ids, tiers)

trainer.train_epoch()
    for batch in dataloader:
        graphs_batch, tokens_batch, y, _, _ = batch
        logits, aux = model(graphs_batch.x, graphs_batch.edge_index,
                            graphs_batch.batch, graphs_batch.edge_attr,
                            tokens_batch["input_ids"],
                            tokens_batch["attention_mask"])
```

---

## §0.6 Exercises (Q1–Q4)

**Q1: The dataset has 22,356 labeled contracts but only 20,100 have graph representations. What does `__len__` return?**

20,100 — the filtered list (line 125: `[sha for sha in all_ids if sha in shard_index]`). The missing 2,256 contracts have labels but no .pt file. They're excluded from training. This is by design — the parquet tables contain all contracts; the dataset only serves those with both modalities.

**Q2: Why does `__getitem__` compute `attention_mask` from `input_ids != PAD_TOKEN_ID` instead of reading from the export?**

The export stores only `tokens.pt` (input_ids). The attention mask is deterministically computable from input_ids (PAD_TOKEN_ID=1 is RoBERTa's pad token). Storing both would double the export size. Computing on-the-fly saves 50% token storage.

**Q3: Two contracts from different shards are in the same batch. How many shard loads happen?**

2 graph shard loads + 2 token shard loads. The LRU cache keeps them for subsequent batches in the same epoch. Next epoch: cache is cold (epoch boundaries `gc.collect()` + `empty_cache()` from Fix #27).

**Q4: The artifact hash verification fails. What should the operator do?**

Re-run `chunk_export()` — the artifact data files are corrupted or tampered. The `verify_artifact_hash()` method provides the warm path (mtime+size cache) and cold path (full SHA-256). If the cold path also fails, the data is unrecoverable — re-export from the split JSONL.
