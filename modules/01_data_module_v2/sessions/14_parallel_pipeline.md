# Session 14 — Parallel Pipeline (`preprocessing/parallel.py`)

**Date:** 2026-06-14
**Session number:** 14 of 36
**Mode:** Understand (156 lines — multiprocessing wrapper with picklable worker, best-effort dedup)
**Estimated study time:** 30 min teaching + 12 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **Serial PreprocessingPipeline** (Session 08): `PreprocessingPipeline.run()` is a single-threaded for-loop iterating `_process_one()` per file.
- **Deduplicator statefulness** (Session 09): The `_dedup` member accumulates SHA-256s across all files. This state DOES NOT survive across parallel workers.
- **`_process_one`** (Session 08 lines 112–216): The core method that runs the 5 steps. It's a private method but `parallel.py` accesses it directly.

---

## §1 The Problem — 22,330 Files at ~1s Each

From the docstring (lines 3–6):

> DIVE = 22,330 files, Web3Bugs = ~3,500, SmartBugs Wild = 47K

At an average of 1 second per file (solc subprocess dominates), serial processing of DIVE takes ~6.2 hours. With an 8-worker pool, this drops to ~45 minutes — nearly 8x speedup (less than linear because of the GIL-free I/O wait overlap and pool overhead).

---

## §2 The Worker Architecture

```
Parent Process
│
├── Creates mp.Pool(8)
│
├── For each sol_file → args = (source_name, out_dir, sol_path, raw_base)
├── pool.imap(_process_one_worker, args_iter, chunksize=174)
│
│   ┌────────────────────────────────────────┐
│   │  Worker 1          Worker 2       ... │
│   │  pipeline =        pipeline =          │
│   │  Preprocessing-    Preprocessing-      │
│   │  Pipeline(...)     Pipeline(...)       │
│   │  _process_one()    _process_one()      │
│   │                     _process_one()     │
│   └────────┬───────────────────────────────┘
│            │  returns dict via pool.imap
│            ▼
├── Parent aggregates results:
│   ├── status="ok"   → parent writes meta.json
│   ├── status="drop" → parent appends dropped row
│   ├── status="error" → parent appends to errors (→ dropped.csv)
│   └── status="skip" → silently discarded
│
└── PipelineResult (processed, dropped, duration_s)
```

### `_process_one_worker` (lines 49–69)

```python
def _process_one_worker(args: tuple) -> dict:
    source_name, out_dir_str, sol_path_str, raw_base_str = args
    pipeline = PreprocessingPipeline(source_name, Path(out_dir_str))
    sol_path = Path(sol_path_str)
    raw_base = Path(raw_base_str)
    try:
        outcome = pipeline._process_one(sol_path, raw_base)
    except Exception as e:
        return {"status": "error", "path": str(sol_path), "error": repr(e)}
    ...
```

**Why module-level?** `mp.Pool` worker functions must be picklable. Bound methods (`pipeline._process_one`) and lambdas aren't. A module-level function IS picklable.

**Why fresh pipeline per worker?** `PreprocessingPipeline` contains a stateful `Deduplicator` (the `_seen_sha`, `_seen_addr`, `_seen_norm` dicts). If two workers shared the same deduplicator via a multiprocessing `Manager`, the dict access would need locks — defeating the parallelism. Instead, each worker has its OWN deduplicator state. The consequence: best-effort dedup (see §3).

---

## §3 Best-Effort Dedup (lines 22–25)

> Two workers might both process the same file in the first 10K before either has recorded the SHA. We accept this — the dedup is best-effort, not exact.

**What can go wrong:**

| Scenario | Effect |
|---|---|
| Worker 1 and 2 both process the same file before either registers the SHA | Both write `<sha256>.sol` and `<sha256>.meta.json` to the same out_path. The later write overwrites the earlier one. **No corruption** (same content → same file) |
| Worker 1 and 2 both process HASH-COLLIDING files (shouldn't happen with SHA-256) | Both write to same path. **Collision undetected**, output overwritten |

**Why this is acceptable:**
- The output filename is content-addressed (SHA-256). Two workers processing the same file produce IDENTICAL output — the last write wins but there's no data loss.
- The meta.json written by the parent (line 126–127) uses the worker's returned meta. If both workers processed the same file, both return identical meta (same source, same compilation result). The last write wins but meta is identical.
- Dropped.csv rows aren't deduplicated — the same file might appear twice in dropped.csv from different workers. But the retry-failed merge is by `original_path`, so duplicates are collapsed.

---

## §4 Parent-Side Meta Writing (lines 122–128)

```python
if status == "ok":
    from sentinel_data.preprocessing.pipeline import _write_meta
    _write_meta(Path(result["out_path"]).with_suffix(".meta.json"), result["meta"])
    processed.append(Path(result["out_path"]))
```

**Why parent writes meta.json instead of the worker:**
- Workers share no filesystem state with each other (by design)
- If two workers both wrote meta.json to the same path, the write isn't atomic — a reader might see a partial write
- The parent process writes meta.json AFTER the worker's result is received, guaranteeing serialized writes
- The worker returns the `ContractMeta` object as part of the result dict (pickled through the pool's result queue)

---

## §5 Chunksize Auto-Tuning (lines 72–77, line 113)

```python
def _chunksize(total: int, n_workers: int) -> int:
    return max(1, total // (n_workers * 16))
```

For 22,330 files with 8 workers: `22330 // (8 * 16) = 22330 // 128 = 174`.

Each worker gets ~174 files in a chunk. The pool distributes chunks: worker 1 gets files 0–173, worker 2 gets 174–347, etc. When a worker finishes its chunk, it grabs the next unassigned chunk.

Larger chunks = less IPC overhead (fewer `pickle`/`unpickle` cycles). Smaller chunks = better load balancing (workers finish at similar times). The `* 16` factor creates 16 chunks per worker, giving the pool enough granularity to balance loads without excessive IPC overhead.

---

## §6 Error Handling (lines 60–63, 139–150)

If a worker raises ANY exception (not just compile failures — also `OSError`, `MemoryError`, etc.), the error is captured and the worker continues processing other files in its chunk:

```python
try:
    outcome = pipeline._process_one(sol_path, raw_base)
except Exception as e:
    return {"status": "error", "path": str(sol_path), "error": repr(e)}
```

The parent aggregates errors and writes them to dropped.csv with `reason: "worker_exception"`:

```python
dropped.append({
    "source": pipeline.source_name,
    "original_path": e["path"],
    "pragma": "",
    "reason": "worker_exception",
    "error": e["error"][:300],
})
```

This is a **fail-safe** — a single corrupted file doesn't kill the entire 22K run. The corrupted file goes to dropped.csv, the pool continues, and the run completes with a report of what failed.

---

## §7 Pipeline Integration (`preprocess.py:90–96`)

```python
if n_workers and n_workers > 1:
    from sentinel_data.preprocessing.parallel import run_preprocess_parallel
    result = run_preprocess_parallel(
        pipeline, sol_files, raw_base=raw_dir, n_workers=n_workers
    )
else:
    result = pipeline.run(sol_files, raw_base=raw_dir)
```

The serial and parallel paths share the same `preprocess_source()` entry point. The switch is a single `if n_workers > 1`. Default is 1 (serial, from `cli.py`). The CLI has a `--workers N` flag to enable parallelism.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — `pool.imap` vs `pool.map`: imap returns results lazily as they complete. What would change if we used map?

`pool.map` blocks until ALL results are ready, then returns them in input order. For 22K files, the parent would sit idle for 45+ minutes, then receive all results at once — no progress feedback, no streaming writes.

`pool.imap` yields results AS THEY COMPLETE. The parent loop writes meta.json progressively. If the process crashes after 30 minutes, you've already written 50% of the meta.json files instead of 0%.

Additionally, the `dropped.csv` append happens incrementally in the imap loop (line 129–130). With `pool.map`, all dropped rows would be accumulated in memory before writing — higher peak memory usage for large runs.

### Q2 — `_process_one_worker` creates a pipeline with `source_name` and `out_dir_str`. The serial `_process_one` expects a `raw_base` argument. How does the worker know `raw_base`?

`raw_base` is passed as the fourth element in the `args` tuple (line 110). The worker reconstructs it as `Path(raw_base_str)` and passes it to `pipeline._process_one(sol_path, raw_base)`. This is the same `raw_dir` that the serial pipeline receives — used for computing `original_path = sol_path.relative_to(raw_base)`.

### Q3 — The chunksize formula `total // (n_workers * 16)` divides by 16. For a 10-file test run with 4 workers: `10 // (4*16) = 10 // 64 = 0`. The `max(1, 0)` gives chunksize=1. Each chunk is 1 file. Good.

For 10 files with 100 workers (unlikely but possible): `10 // (100*16) = 0` → chunksize=1. Each chunk is 1 file, and workers 4–100 get no work. This is fine — the pool only assigns as many chunks as there are files.

For 22,330 files with 100 workers (on a server-class machine): `22330 // 1600 = 13`. Each chunk is 13 files. With 100 workers × 16 chunks each = 1717 total chunks. 1717 × 13 = 22330 — exact fit. Load balancing is optimal.

### Q4 — Each worker creates its own `PreprocessingPipeline` and thus its own `Deduplicator`. What's the practical impact on dedup rate vs. the serial pipeline?

The serial pipeline has a single deduplicator that accumulates all SHA-256s across the entire run. The parallel pipeline has N independent deduplicators (one per worker). A file that is a duplicate of a file processed by a DIFFERENT worker will NOT be caught — both workers would process it independently, and both would write its output file.

**How many duplicates slip through?** Depends on file distribution across workers. If duplicates are clustered in the dataset (e.g., the same file in adjacent class folders), they're likely processed by the same worker (same chunk) and caught. If duplicates are scattered, they cross worker boundaries and slip through.

**How does this affect the final dataset?** Slightly inflated (more files than the serial pipeline would produce). The SHA-256-based dedup in the splitting stage (Stage 5) catches cross-file duplicates later, so no final-label noise is introduced — only a small preprocessing-time inflation. The pipeline designates this as acceptable.
