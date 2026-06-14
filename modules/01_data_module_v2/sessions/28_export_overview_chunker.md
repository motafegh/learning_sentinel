# Session 28 — Export Overview + Chunker (Stage 7A)

**Date:** 2026-06-14
**Session number:** 28 of 36
**Mode:** Understand (chunker.py 237L, export.py 173L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Splits** (Session 25): train/val/test JSONL files with contract order
- **Graph reps** (Session 16): `.pt` files per contract in `data/representations/`
- **Tokens** (Session 20): `.tokens.pt` per contract, shape [4, 512]
- **Merged labels** (Session 27): `.labels.json` per contract in `data/labels/merged/`

---

## §1 What Export Produces

```
<output_dir>/
├── labels.parquet           # 14 columns, all 22K labeled contracts
├── metadata.parquet         # 14 columns, enriched from sidecars
├── graphs/
│   ├── graphs-00000.pt      # sharded PyG Batch, 5K contracts/shard
│   ├── graphs-00001.pt
│   └── _shard_index.json    # sha256 → (shard, pos_in_shard, num_nodes)
├── tokens/
│   ├── tokens-00000.pt      # sharded [N, 4, 512] tensor
│   ├── tokens-00001.pt
│   └── _shard_index.json
└── manifest.json            # written LAST (Fix A)
```

---

## §2 Chunker — The Orchestrator (237 LOC)

```python
def chunk_export(
    rep_root,           # data/representations
    splits_dir,         # data/splits/<run_id>/
    labels_root,        # data/labels/merged
    output_dir,         # data/exports/<version>/
    shard_size=5000,
) -> ExportManifest
```

Runs 4 writers in order:

1. `write_labels_parquet()` — writes `labels.parquet`
2. `write_metadata_parquet()` — writes `metadata.parquet`
3. `write_graphs_shards()` — writes `graphs/*.pt`
4. `write_tokens_shards()` — writes `tokens/*.pt`
5. Computes `artifact_hash` over all data files
6. Writes `manifest.json` **LAST** (Fix A — manifest is excluded from the hash to avoid circular hash)

### Fix A — Why Write Manifest Last

The `artifact_hash` is SHA-256 over all data file paths + contents (lines 65–79):

```python
_HASH_EXCLUDED = {"manifest.json", ".hash_cache.json"}

def _hash_export_data(export_dir):
    candidate_files = sorted(p for p in export_dir.rglob("*")
                             if p.is_file() and p.name not in _HASH_EXCLUDED)
    h = hashlib.sha256()
    for p in candidate_files:
        rel = str(p.relative_to(export_dir))
        h.update(rel.encode())
        h.update(p.read_bytes())
    return h.hexdigest()
```

If manifest were written before hashing, any schema version change would change the hash → broken integrity chain. Writing LAST means the manifest reflects the hash that excludes itself.

---

## §3 SentinelDatasetExport — Consumer API (173 LOC)

```python
class SentinelDatasetExport:
    """Read-only view of an export directory."""

    def __init__(self, export_dir):
        self.export_dir = Path(export_dir)
        self.manifest = self._parse_manifest(self._manifest_raw)

    @property
    def graphs_dir(self) -> Path       # export_dir / "graphs"
    @property
    def tokens_dir(self) -> Path       # export_dir / "tokens"
    @property
    def labels_path(self) -> Path      # export_dir / "labels.parquet"
    @property
    def metadata_path(self) -> Path    # export_dir / "metadata.parquet"
    @property
    def shard_index(self) -> dict      # sha256 → shard metadata

    def verify_artifact_hash(self) -> bool:
        """Compare artifact hash to manifest.artifact_hash.
        Warm path: uses .hash_cache.json (mtime + size, no disk reads).
        Cold path: full SHA-256 over all data files.
        """
```

This is the bridge to the ML side. The model training `SentinelDataset` wraps this class to load shards by index.

---

## §4 Shard Index Format

```json
{
  "abc123...": {"shard": 0, "pos_in_shard": 0, "num_nodes": 42},
  "def456...": {"shard": 0, "pos_in_shard": 1, "num_nodes": 17},
  ...
}
```

The model trainer uses `shard_index[contract_id]` to find the right shard file and slice the right graph from the batch.

---

## §0.6 Exercises (Q1–Q4)

**Q1: A contract has a label but no graph representation. Does it appear in the export?**

Yes — it appears in `labels.parquet` and `metadata.parquet` (with null graph fields). It does NOT appear in the graph shards (skipped by `write_graphs_shards`). The parquet tables have all 22K+ labeled contracts; the graph shards only cover contracts with reps.

**Q2: Someone modifies a graph shard after export. What detects this?**

`verify_artifact_hash()` — warm path detects changed mtime/size via `.hash_cache.json`. Cold path recomputes SHA-256 and compares to `manifest.artifact_hash`. If mismatch, returns False.

**Q3: Why is `_shard_index.json` excluded from the artifact hash?**

It's in the `graphs/` directory, so `rglob("*")` would include it. But `_HASH_EXCLUDED = {"manifest.json", ".hash_cache.json"}` only excludes files by name. The shard index IS part of the hash (it's a data file). If it's tampered with, the hash breaks.

**Q4: The `SentinelDatasetExport` is for model training. What does the ML wrapper add?**

`__len__` (contract count from manifest), `__getitem__` (load shard → slice graph/token at `pos_in_shard` → return `(graph, tokens, label_vector, metadata)` tuple), collation into minibatches.
