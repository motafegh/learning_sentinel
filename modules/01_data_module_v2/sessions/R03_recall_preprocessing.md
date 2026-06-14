# Recall R3 — Preprocessing Pipeline (Sessions 08–11)

**Purpose:** Recall, review, relearn — consolidate 4 sessions into one
**Estimated review time:** 25 min
**Difficulty:** Core (must know for interviews)

---

## 1. The Big Picture

Preprocessing transforms raw `.sol` files into ML-ready canonical form. The `PreprocessingPipeline` runs **5 steps** per file: flatten → compile → dedup → normalize → segment+bucket. Output goes to `data/preprocessed/<source>/<sha256>.sol` + `<sha256>.meta.json`. Failed files go to `dropped.csv`. The orchestrator (`preprocess.py`) supports 3 modes: full, sample, retry-failed.

---

## 2. Architecture Diagram

```
preprocess.py (service orchestrator)
  │
  └─ preprocess_source(name, cfg, data_dir, ...)
       │
       ├── 1. Verify manifest exists
       │
       ├── 2. Build file list
       │     ├── --retry-failed → _load_dropped_files()
       │     └── full           → manifest["files"]
       │         └── --sample N → first N from manifest
       │
       ├── 3. _maybe_folderize()           ← Session 07 bridge
       │
       └── 4. PreprocessingPipeline.run()
              │
              │  For each file:
              │
              │  Step 1: flatten_contract()     ← Flattener (Session 11)
              │    │   Resolves imports → single self-contained .sol
              │    │   Falls back to stripping unresolvable imports
              │    │
              │  Step 2: compile_contract()      ← Compiler (Session 10)
              │    │   Two-pass: exact version → nearest satisfying
              │    │   Temp file protocol if flattened ≠ original
              │    │   FAIL → dropped.csv ("compile_failed")
              │    │
              │  Step 3: Deduplicator.process()  ← Deduplicator (Session 09)
              │    │   3 levels: SHA-256 → address → normalized text
              │    │   Stateful: grows _seen_sha across entire run
              │    │   DUP  → dropped.csv ("duplicate")
              │    │
              │  Step 4: normalize()             ← Normalizer (Session 12)
              │    │   Strip SPDX, comments, trailing whitespace
              │    │
              │  Step 5: segment_and_bucket()    ← Segmenter (Session 13)
              │         Version bucket, contract names, unchecked
              │
              └── Write <sha256>.sol + .meta.json
```

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| 5-step pipeline order and WHY | 08 | ☐ |
| `ContractMeta` — 16 fields as audit trail | 08 | ☐ |
| `DroppedContract` vs `preprocessed/` | 08 | ☐ |
| Temp file protocol (`.sentinel_compile_`, `sentinel_stripped`) | 08 | ☐ |
| 3-level dedup: SHA → address → normalized | 09 | ☐ |
| Why identifier lowercasing is NOT applied | 09 | ☐ |
| Two-pass compile: exact then satisfying | 10 | ☐ |
| `--allow-paths` and why temp files stay in source dir | 10 | ☐ |
| `_pick_solc` oldest-first vs compiler newest-first | 11 | ☐ |
| `_ASSUMED_BARE_IMPORT_SYMBOLS` for forge-std | 11 | ☐ |
| Recursive strip + transitive `.sentinel_stripped.sol` | 11 | ☐ |
| Retry-failed mode — incremental build pattern | 08 | ☐ |
| `_merge_retry_results` — keyed by `original_path` | 08 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: Explain the 5-step pipeline order and why each step is where it is.**

1. **Flatten first** — compilation (step 2) needs a single self-contained file
2. **Compile before dedup** — we only dedup compilable files; dedup on flattened content catches more duplicates than raw text
3. **Dedup before normalize** — no point normalizing a duplicate that will be dropped
4. **Normalize before segment** — segment is purely analytical, runs on the final cleaned text

**Q2: Why does `_merge_retry_results` key on `original_path` instead of `sha256`?**

Dropped files never reach Step 3 (dedup), so they have NO sha256. The sha256 is computed by the Deduplicator on the flattened content. A compile failure (Step 2) happens before dedup runs. The only stable identifier in `dropped.csv` is `original_path`.

**Q3: How does the temp file protocol prevent import resolution failures?**

When the flattener modifies content (imports resolved), the pipeline writes a temp `.sentinel_compile_` file RIGHT NEXT to the original (same directory). This ensures relative imports like `../interface.sol` resolve correctly — they would fail from `/tmp/`. The temp file is cleaned in a `try/finally` block.

**Q4: The 3-level dedup has a known transitivity bug. What is it?**

Level 2 (address check) fires on the first address match and returns early BEFORE registering the file's OTHER addresses in `_seen_addr`. So file C that shares an address with file B (which shares with file A) won't be caught by Level 2 if C's address was only introduced by B and B was already flagged as a duplicate. Fix: register ALL addresses before checking any.

---

## 5. Session-by-Session Memory Triggers

- **S08:** Pipeline overview, `ContractMeta`, `PipelineResult`, 5 steps, temp file protocol, retry-failed merge
- **S09:** 3-level dedup, `_normalize_for_dedup`, stateful `_seen_*` dicts, no identifier lowercasing
- **S10:** Two-pass compile, `_parse_version`, `_satisfying_versions`, `_run_solc --allow-paths`, `_available_versions`
- **S11:** `flatten_contract`, solc --flatten fallback, `_strip_unresolved_imports_recursive`, inheritance strip, transitive strip

---

## 6. What To Remember

1. **SHA-256 is the identity** — output files named by content hash, not contract name.
2. **The manifest is the gate** — no manifest → no preprocessing.
3. **Retry-failed is build-system incremental** — fix a version → retry only dropped files.
4. **Dedup is best-effort in parallel mode** — each worker has its own dedup state.
5. **Flattener is the most complex sub-module** — cascading fallbacks for unresolvable imports.
6. **Temp file protocol** ensures import resolution + deterministic cleanup.
