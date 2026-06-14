# Session 08 — The Preprocessing Engine

**Date:** 2026-06-14
**Session number:** 08 of 36
**Mode:** Understand (core pipeline — serial 5-step per-file processing)
**Estimated study time:** 60 min teaching + 20 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know (prerequisites)

- **Ingestion manifest** (`ingestion_manifest.json`): you know it records the file list with `include_subdirs`/`exclude_subdirs` scoping (Session 05 — connectors)
- **Label folderize** (`label_folderize.py`): you know `folderize_by_labels` creates per-class symlinks (Session 07)
- **CLI router** (`cli.py`): you know `_run_preprocess` dispatches to `preprocess_source` or `preprocess_all` (Session 04)
- **Config structure**: you know `sources.<name>` entries with `enabled`, `extra`, `labels_csv`, etc.

---

## §1 `pipeline.py` — The 5-Step `PreprocessingPipeline`

**File:** `sentinel_data/preprocessing/pipeline.py` (246 LOC)
**Design pattern:** Single-pass serial pipeline per file. The `PreprocessingPipeline` class owns the 5 steps and accumulates results into a `PipelineResult`. Each file gets a `ContractMeta` sidecar JSON.

### §1.1 `ContractMeta` — The Audit Trail (lines 33–60)

A contract processed by the pipeline has **16 tracked fields**:

| Field | Source | Meaning |
|---|---|---|
| `sha256` | Deduplicator | Content hash → filename |
| `source_name` | CLI arg | e.g., `"defihacklabs"` |
| `original_path` | `sol_path.relative_to(raw_base)` | Traceback to raw file |
| `pragma` | Compiler | Raw `pragma solidity ^0.8.0;` text |
| `solc_version` | Compiler | Resolved version e.g., `"0.8.17"` |
| `compile_status` | Compiler | `"ok"` or `"failed"` |
| `compile_error` | Compiler | Truncated error (300 chars) |
| `attempted_solc_versions` | Compiler | All versions tried |
| `flatten_status` | Flattener | `"flattened"`, `"skipped_no_imports"`, `"skipped_error"` |
| `dedup_group_id` | Deduplicator | Cluster group |
| `is_duplicate` | Deduplicator | Was this a duplicate? |
| `duplicate_of` | Deduplicator | SHA of original |
| `version_bucket` | Segmenter | `"legacy"`, `"transitional"`, `"modern"` |
| `has_unchecked_block` | Segmenter | Uses `unchecked { }`? |
| `contract_names` | Segmenter | All contract/interface/library names |
| `n_raw_lines` / `n_normalized_lines` | Pipeline | Line count before/after |

Plus `meta_schema_version: str = "1"` and `extra: dict[str, Any]` for forwards-compatibility.

**Key insight:** The meta.json IS the canonical record. Every downstream stage (labeling, verification, splitting) reads meta.json to know what happened during preprocessing. If the meta.json is missing, the contract is invisible to Stage 2.

### §1.2 `PipelineResult` (lines 63–69)

```python
@dataclass
class PipelineResult:
    processed: list[Path]     # Output .sol paths written
    dropped: list[dict]       # Rows for dropped.csv
    duration_s: float
```

Aggregate output. The `processed` paths are `out_dir/<sha256>.sol`. The `dropped` rows become CSV rows. Duration is wall-clock.

### §1.3 `PreprocessingPipeline` — The Orchestrator (lines 72–110)

```python
class PreprocessingPipeline:
    def __init__(self, source_name: str, out_dir: Path):
        self.source_name = source_name
        self.out_dir = out_dir
        self._dedup = Deduplicator()       # Stateful: tracks all SHA-256s seen so far
```

**Statefulness:** The `_dedup` member is NOT reset between files. This is critical — deduplication is a SET over the entire source run. File #7,500 might be a duplicate of file #312. The set grows monotonically.

**`run()` method (lines 80–110):**
```python
def run(self, sol_files: list[Path], raw_base: Path) -> PipelineResult:
    # 1. Create out_dir
    # 2. For each sol_path:
    #      outcome = self._process_one(sol_path, raw_base)
    #      if outcome is None: continue  (dead code — see Q1)
    #      out_sol, meta, drop_row = outcome
    #      if drop_row: append to dropped
    #      else:         write meta.json + append to processed
    # 3. If dropped non-empty: write dropped.csv
```

The `run()` loop is a simple for-loop. No multiprocessing (see §1.4 for why).

### §1.4 `_process_one` — The 5 Steps (lines 112–216)

```
     ┌──────────┐
     │  .sol    │
     │  file    │
     └────┬─────┘
          │
          ▼
    ┌──────────┐
    │  1.      │  flatten_contract(sol_path)
    │ FLATTEN  │  → FlattenResult(content, flatten_status)
    └────┬─────┘     Resolves imports, produces single-file version
         │
         ▼
    ┌──────────┐
    │  2.      │  compile_contract(compile_target)
    │ COMPILE  │  → CompileResult(success, solc_version, evm_bytecode, ...)
    └────┬─────┘     Two-pass: first with pragma version, fallback scan
         │
    ┌────▼─────┐
    │ FAILED?  │──→ dropped.csv row ("compile_failed")
    └────┬─────┘
         │ (success)
         ▼
    ┌──────────┐
    │  3.      │  Deduplicator.process(flat.content, sol_path)
    │  DEDUP   │  → DedupRecord(sha256, is_duplicate, duplicate_of, dedup_group_id)
    └────┬─────┘     Compares SHA-256 against all previous files
         │
    ┌────▼─────┐
    │DUP?      │──→ dropped.csv row ("duplicate")
    └────┬─────┘
         │ (unique)
         ▼
    ┌──────────┐
    │  4.      │  normalize(flat.content)
    │NORMALIZE │  → NormalizeResult(content, n_lines_after)
    └────┬─────┘     Strips SPDX, metadata hash, normalizes whitespace
         │
         ▼
    ┌──────────┐
    │  5.      │  segment_and_bucket(norm.content, pragma_raw)
    │SEGMENT+  │  → SegmentResult(version_bucket, has_unchecked_block, contract_names)
    │  BUCKET  │     Classifies into legacy/transitional/modern
    └──────────┘
         │
         ▼
     out_dir/<sha256>.sol   +   .meta.json
```

**Step details:**

1. **Flatten** (line 121): Calls `flatten_contract(sol_path)` which resolves `import` statements and produces a single self-contained `.sol` string. If no imports exist, `flatten_status = "skipped_no_imports"` and content = source text.

2. **Compile** (lines 130–173): The most complex step. Uses `compile_contract(compile_target)` which shells out to `solc` via `solc-select`. **Temp file protocol** (lines 130–163):
   - If the flattener modified content (`flat.content != source`), writes a temp file at `sol_path.parent / ".sentinel_compile_{stem}_{mtime_ns}.sol"`.
   - Compiles the temp file (NOT the original) so relative imports resolve.
   - In a `try/finally`, cleans up the temp file AND any `.sentinel_stripped.sol` sibling files written by the flattener's transitive-strip helper.
   - If compile fails, returns a `dropped` dict with `reason: "compile_failed"` and the error truncated to 300 chars.

   **Why not `/tmp`?** Solidity imports like `import "../interfaces/ISomething.sol"` are RELATIVE to the file's directory. A temp file in `/tmp` would break all relative imports.

3. **Dedup** (lines 176–185): `Deduplicator.process(flat.content, sol_path)` computes SHA-256 of the flattened content and checks against all previous SHA-256s from this pipeline run. If a match is found, `is_duplicate = True` and returns a dropped dict with `reason: "duplicate"`.

4. **Normalize** (line 188): `normalize(flat.content)` strips SPDX license identifiers, empty comment blocks, trailing whitespace, and metadata hash references. Output is a clean normalized source string.

5. **Segment + Bucket** (line 191): `segment_and_bucket(norm.content, pragma_raw)` classifies the contract:
   - `version_bucket`: `"legacy"` (<0.5.0), `"transitional"` (0.5.0–0.7.6), `"modern"` (≥0.8.0)
   - `has_unchecked_block`: Boolean — does it use Solidity 0.8's `unchecked { }`?
   - `contract_names`: Extracts all `contract`, `interface`, `library`, `abstract contract` names

**Output writing** (lines 193–216): The output file is named by SHA-256: `out_dir / f"{dedup_rec.sha256}.sol"`. The `ContractMeta` is assembled from all step results and written as `out_dir / f"{dedup_rec.sha256}.meta.json"` via `_write_meta()`.

### §1.5 `_write_meta` / `_write_dropped` (lines 219–246)

- `_write_meta`: `json.dump(asdict(meta), f, indent=2)` — the sidecar.
- `_write_dropped`: Union-of-all-keys approach (defensive: compile_failed has 5 fields, duplicate has 6). Normalizes missing keys to `""`. Writes `dropped.csv` with a header row.

## §2 `preprocess.py` — The Orchestrator (264 LOC)

**File:** `sentinel_data/preprocessing/preprocess.py`
**Role:** CLI-facing service that selects & validates a source, then runs the pipeline. Parallel to `ingest.py` in the Session 04 architecture.

### §2.1 `preprocess_source()` — 3 Modes (lines 29–107)

**Signature:**
```python
def preprocess_source(
    name: str, cfg: dict, data_dir: Path,
    dry_run: bool = False, n_workers: int = 1,
    sample: int | None = None, retry_failed: bool = False,
) -> None:
```

**Flow:**

```
1. Validate source is enabled in config
2. Resolve paths:
     raw_dir  = data_dir / "raw" / name
     out_dir  = data_dir / "preprocessed" / name
3. Verify manifest exists (gate — see §2.5)
4. Build file list:
   a. retry_failed=True  → _load_dropped_files()
   b. retry_failed=False → read manifest["files"] → optionally apply --sample N
5. dry_run=True  → print info + return
6. _maybe_folderize()  (full mode only — see §2.3)
7. Instantiate PreprocessingPipeline
8. n_workers > 1 → parallel.run_preprocess_parallel()
   n_workers = 1 → pipeline.run(sol_files, raw_base=raw_dir)
9. retry_failed=True → _merge_retry_results()
   else → dropped.csv already written by pipeline.run()
```

**Mode details:**

| Mode | Trigger | File selection | folderize | Output merging |
|---|---|---|---|---|
| Full | `sentinel-data preprocess --source X` | All manifest files | Yes | New dropped.csv (overwrite) |
| Sample | `--sample 500` | First 500 from manifest | Yes | Same as full |
| Retry-failed | `--retry-failed` | Only files in existing `dropped.csv` | No (already set up) | MERGE with old dropped.csv |

### §2.2 `preprocess_all()` — Source Iterator (lines 247–264)

```python
def preprocess_all(cfg, data_dir, dry_run=False, n_workers=1, sample=None, retry_failed=False):
    for name in _enabled_sources(cfg):
        print(f"\n[preprocess] {name}")
        try:
            preprocess_source(name, cfg, data_dir, ...)
        except FileNotFoundError as e:
            print(f"  SKIP — {e}")
```

Wraps per-source calls. Skips if manifest missing (no raw data for that source yet). Called by `sentinel-data preprocess --all`.

### §2.3 `_maybe_folderize()` — The Session 07 Bridge (lines 200–244)

**`preprocess.py:200`** is where Session 07 connects to Session 08.

```python
def _maybe_folderize(raw_dir: Path, entry: dict, name: str) -> None:
    labels_csv_rel = entry.get("labels_csv")
    if not labels_csv_rel:
        return           # No labels → no folderization (most sources)
    ...
    folderize_by_labels(
        repo_dir=raw_dir / "repo",
        labels_csv=labels_csv,
        id_column=entry.get("label_id_column", "contractID"),
        class_columns=entry.get("class_columns") or [],
        source_subdir=entry.get("label_source_subdir", "__source__"),
    )
```

This is called BEFORE the pipeline runs (line 86–87 in full mode), because the pipeline's `find_sol_files()` (used internally by the compile step's import resolver) relies on the per-class symlink layout.

**Config entry example that triggers folderization:**
```yaml
sources:
  dive:
    enabled: true
    type: git
    extra:
      labels_csv: labels/dive.csv
      label_id_column: contractID
      class_columns: ["is_reentrancy", "is_access_control"]
```

### §2.4 `_load_dropped_files()` + `_merge_retry_results()` (lines 110–197)

These implement the **incremental build** pattern:

**`_load_dropped_files`** (lines 110–132):
- Reads `out_dir/dropped.csv`
- Reconstructs file paths as `raw_dir / row["original_path"]`
- Silently skips paths that no longer exist on disk (file removed during re-ingest)
- Returns `(paths, rows)` — the source paths AND the raw CSV rows for later merging

**`_merge_retry_results`** (lines 135–197):
- After the retry pipeline runs, reads each newly-written `.meta.json` to discover which `original_path`s now succeed
- Builds a merge keyed by `original_path`:
  - **Was in dropped.csv, now succeeds** → remove from dropped.csv
  - **Was in dropped.csv, still fails** → overwrite error with NEW error message
  - **Was in dropped.csv, not retried** (file removed) → preserve old row
- If all retries succeeded, `dropped.csv` is deleted entirely (line 195–197)

**Why key on `original_path` not `sha256`?** Because dropped files never reach Step 3 (dedup), so they have NO sha256. The `original_path` is the only stable identifier in `dropped.csv`.

### §2.5 Manifest Verification Gate (lines 52–56)

```python
manifest_path = raw_dir / "ingestion_manifest.json"
if not manifest_path.exists():
    raise FileNotFoundError(
        f"Ingestion manifest {manifest_path} not found. "
        f"Run `sentinel-data ingest --source {name}` first."
    )
```

This is the **hard gate** between ingestion and preprocessing. No manifest → no preprocessing. The manifest IS the contract between ingest and preprocess — it records exactly which files were ingested, with which scoping. Preprocessing NEVER re-globs the raw directory.

### §2.6 Parallel Mode (lines 90–94)

```python
if n_workers and n_workers > 1:
    from sentinel_data.preprocessing.parallel import run_preprocess_parallel
    result = run_preprocess_parallel(pipeline, sol_files, raw_base=raw_dir, n_workers=n_workers)
else:
    result = pipeline.run(sol_files, raw_base=raw_dir)
```

The parallel module (`preprocessing/parallel.py`) wraps `PreprocessingPipeline` in a multiprocessing `Pool`. Default is serial (`n_workers=1`). The design doc explicitly defers full multiprocessing to v2.1 (see docstring lines 8–11): `solc` subprocesses already saturate I/O, so the gain is small for a one-time pipeline.

---

## §3 Architectural Summary

```
                    ┌─────────────────────────────────────┐
                    │            CLI (cli.py)              │
                    │   sentinel-data preprocess [--source]│
                    └──────────┬──────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  preprocess.py      │  service orchestrator
                    │                     │
                    │  preprocess_source()│  3 modes (full, sample, retry-failed)
                    │  preprocess_all()   │  iterates enabled sources
                    │  _maybe_folderize() │  Session 07 bridge
                    │  _merge_retry_results│  incremental build
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   pipeline.py        │  core pipeline
                    │                      │
                    │   PreprocessingPipe- │  stateful (dedup set)
                    │   line.run()         │
                    │                      │
                    │   1. flatten         │  resolve imports
                    │   2. compile         │  solc subprocess
                    │   3. dedup           │  SHA-256 set
                    │   4. normalize       │  strip metadata
                    │   5. segment+bucket  │  classify version
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Output              │
                    │                      │
                    │  data/preprocessed/  │
                    │    <source>/         │
                    │      <sha256>.sol    │  normalised source
                    │      <sha256>.meta.json  audit trail
                    │      dropped.csv     │  failure log
                    └─────────────────────┘
```

**Key architectural invariants:**

1. **The manifest is the gate.** No manifest → no preprocessing. Preprocessing never re-globs `raw/`.
2. **SHA-256 is the identity.** Output files are named by content hash. Dedup is SHA-256 equality. `dropped.csv` is the exception (keyed by `original_path` because failures have no SHA-256).
3. **Serial execution by design.** Multiprocessing is deferred to v2.1 because `solc` subprocesses already saturate I/O.
4. **Sidecar pattern.** Every `.sol` output has a sibling `.meta.json` that encodes the full 5-step audit trail. No downstream stage reads `.sol` without its meta.
5. **Folderization before pipeline.** `_maybe_folderize()` runs before the pipeline — the pipeline expects per-class symlinks if labels exist.
6. **Incremental retry.** `--retry-failed` implements a build-system-style incremental: fix a compiler version → re-run only dropped files → merge results. No 22,000-file reprocess.

---

## §0.6 Session 08 Exercises (Q1–Q4)

### Q1 — Dead Code or Future-Proofing? (lines 93–95)

```python
outcome = self._process_one(sol_path, raw_base)
if outcome is None:
    continue
out_sol, meta, drop_row = outcome
```

`_process_one` always returns `(Path, ContractMeta, None)` or `(None, None, dict)` — it NEVER returns `None`. So `if outcome is None: continue` is unreachable. Is this dead code, or does it serve a purpose?

### Q2 — Manifest vs Glob (lines 66–73)

`preprocess_source()` reads the manifest file list instead of running `raw_dir.rglob("*.sol")`:
```python
manifest = json.loads(manifest_path.read_text())
for entry in manifest["files"]:
    rel = entry["path"]
    sol_files.append(raw_dir / rel)
```

Why is manifest-based file selection preferred over a fresh recursive glob? What would go wrong if we re-globbed?

### Q3 — `original_path` as Merge Key (lines 173–188)

`_merge_retry_results` merges by `original_path` (a path string like `"repo/buggy_contracts/x.sol"`). Why not use `sha256` as the merge key? And what does this reveal about the contract between `dropped.csv` and the pipeline?

### Q4 — `raw_base` = `raw_dir`, not `raw_dir / "repo"` (line 96)

The `pipeline.run()` call passes `raw_base=raw_dir`:
```python
result = pipeline.run(sol_files, raw_base=raw_dir)
```

Inside `_process_one`, `rel = sol_path.relative_to(raw_base)` — so `original_path` ends up as `"repo/buggy_contracts/x.sol"` (includes the `repo/` prefix). Why NOT pass `raw_dir / "repo"` so `original_path` is `"buggy_contracts/x.sol"` (without the prefix)? Trace how this affects `_load_dropped_files` and explain why the `repo/` prefix matters.

---

**Your answer to Q1:** The `if outcome is None: continue` check is **defensive dead code**. `_process_one` currently never returns `None`, but the guard exists so that a future refactoring can add an early-return `None` path (e.g., "skip empty text files at the flatten step") without crashing `run()`'s tuple unpacking. Removing it wouldn't change behavior today, but it's intentional defensive programming for future skippable-file cases.

**Your answer to Q2:** Manifest-based selection preserves **ingest-time scoping**. The connectors apply `include_subdirs` and `exclude_subdirs` during ingestion, and the manifest records exactly which files passed those filters. Re-globbing `raw_dir/**/*.sol` would: (a) include files outside `include_subdirs`, (b) include files in `exclude_subdirs` (e.g., SolidiFI's `results/` directory with test artifacts), (c) process files added to `raw/` **after** ingestion that weren't part of the original import. The manifest is the canonical "what was intentionally imported" snapshot.

**Your answer to Q3:** Dropped files **never have a SHA-256** because compilation failure happens at Step 2, before the Deduplicator (Step 3) computes the hash. The `dropped.csv` rows are written by the compile step with no `sha256` field. The only stable identifier available is `original_path`. This reveals a fundamental pipeline invariant: **SHA-256 is a post-compile identity, not a pre-compile one.** Before compilation, identity is source location; after compilation + dedup, identity is content hash.

**Your answer to Q4:** Manifest paths include the connector's subdirectory prefix (e.g., `"repo/"` from the Git connector, or `"contracts/"` from a different connector). If `raw_base = raw_dir / "repo"`, then `original_path` would be `"buggy_contracts/x.sol"` (no `repo/` prefix). In `_load_dropped_files`, path reconstruction does `raw_dir / original_path` = `raw_dir / "buggy_contracts/x.sol"` — but the actual file lives at `raw_dir / "repo" / "buggy_contracts/x.sol"`. The path reconstruction FAILS silently (file not found → skipped). By using `raw_base = raw_dir`, `original_path` preserves `"repo/buggy_contracts/x.sol"`, and `raw_dir / original_path` correctly resolves to the real file. The `repo/` prefix is **not redundant** — it encodes the connector's materialization strategy, which varies per source.
