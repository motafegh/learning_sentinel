# Session 29 — Export Writers (graph, label, metadata, token)

**Date:** 2026-06-14
**Session number:** 29 of 36
**Mode:** Understand (graph_writer 106L, label_writer 150L, metadata_writer 200L, token_writer 95L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Export manifest** (Session 28): orchestrator runs 4 writers + computes hash
- **Split JSONL** (Session 25): train/val/test ordered lines with contract data
- **PyG graphs** (Session 16): per-contract `.pt` files
- **Tokens** (Session 20): per-contract `.tokens.pt` files

---

## §1 Label Writer (150 LOC) — `labels.parquet`

```python
def write_labels_parquet(splits_dir, output_dir, split_name_order=("train", "val", "test")):
```

Reads the split JSONL (3 files), writes a single `labels.parquet` with 14 columns:

| Column | Type | Notes |
|---|---|---|
| `contract_id` | string | sha256 |
| `source` | string | e.g. "solidifi" |
| `split` | string | "train"/"val"/"test" |
| `class_0` … `class_9` | int8 | locked taxonomy order |
| `confidence_tier` | string\|null | `null` for NonVulnerable |

**Fix 7A #1:** Reads from split JSONL only — no re-scattering of merged label files. The split JSONL is the canonical input.

**Fix 7A #2:** `confidence_tier` = split tier when `n_pos > 0`, else `None`. The splitter defaults tier='T0' for NonVulnerable; we override to `null` so consumers can filter on `null` vs. actual tier values.

---

## §2 Metadata Writer (200 LOC) — `metadata.parquet`

```python
def write_metadata_parquet(splits_dir, rep_root, preproc_root, output_dir):
```

One row per labeled contract. 14 columns:

| Column | Source |
|---|---|
| `contract_id`, `source`, `split` | split JSONL |
| `solc_version`, `version_bucket` | `.meta.json` sidecar |
| `loc`, `n_functions` | **Computed from .sol source** (split JSONL's `loc=0` is a default placeholder) |
| `n_pos`, `primary_class` | split JSONL |
| `node_count`, `edge_count` | `.rep.json` (graph sidecar) |
| `has_unchecked_block` | `.rep.json` |
| `dedup_group_id` | `.meta.json` |
| `confidence_tier` | same as labels.parquet |

**Fix 7A #3:** `loc` and `n_functions` are computed via `_loc()` and `_function_count()` from `feature_dist.py`. The split JSONL stores `loc=0` as a default — the metadata writer overwrites with the real value.

---

## §3 Graph Writer (106 LOC) — `graphs/*.pt`

```python
def write_graphs_shards(rep_root, splits_dir, output_dir, shard_size=5000):
```

1. Loads ordered contract list from split JSONL (train → val → test)
2. Reads each contract's `.pt` from `rep_root`
3. Batches `shard_size` graphs into a `torch_geometric.data.Batch`
4. Writes `graphs-{shard:05d}.pt`
5. Returns `shard_index`: `{sha256: {shard: int, pos_in_shard: int, num_nodes: int}}`

**Edge case:** Skipped contracts (no `.pt` file) are logged with a warning. They're absent from shards but still in parquet tables.

**PyG safe_globals registration** (line 26):
```python
torch.serialization.add_safe_globals([Data, DataEdgeAttr, DataTensorAttr, GlobalStorage])
```
Required for `weights_only=True` PyTorch serialization.

---

## §4 Token Writer (95 LOC) — `tokens/*.pt`

```python
def write_tokens_shards(rep_root, splits_dir, output_dir, shard_size=5000):
```

Same pattern as graph writer but stacks contract tokens into a `[N, 4, 512]` tensor per shard:

```python
current_tensors: list[torch.Tensor] = []

def _flush():
    batch = torch.stack(current_tensors, dim=0)  # [N, 4, 512]
    torch.save(batch, output_dir / f"tokens-{shard_num:05d}.pt")
```

**Fixed shard size match:** Both graph_writer and token_writer use `shard_size=5000` from the same `chunk_export()` call. Contract order is identical (both use `_load_split_jsonl`). `shard_index[sha]` from graph_writer maps to the same shard number for tokens.

---

## §0.6 Exercises (Q1–Q4)

**Q1: A contract appears at row 7,502 in the split JSONL. Which shard file contains its graph? Which position?**

`shard_size=5000`. Row 7,502 → shard 1 (indices 5000-9999), position 2,501 within the shard. Both graph and token writers place it at the same shard+position because they iterate the same input in the same order.

**Q2: Why does the metadata writer need to recompute `loc` from the .sol source?**

The split JSONL's `loc=0` is a COST-SAVING default — computing LoC during splitting would require reading every `.sol` file at that point. Instead, the split JSONL stores a fast default, and the metadata writer does the expensive computation once during export.

**Q3: A contract's .pt file has `num_nodes=85`. Where is this recorded for the model trainer?**

In the `shard_index` dict returned by `write_graphs_shards()`, stored as `_shard_index.json`. The model trainer reads this to know how many nodes each graph has before loading the shard.

**Q4: What happens if the shard_size is changed from 5000 to 1000 between graph and token writer calls?**

Can't happen — both writers are called from the same `chunk_export()` function with the same `shard_size` parameter. The contract order is identical. The two indexes would match perfectly.
