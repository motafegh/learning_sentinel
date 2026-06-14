# Recall R4 — Preprocessing Remainder + Parallel (Sessions 12–14)

**Purpose:** Recall, review, relearn — consolidate 3 sessions into one
**Estimated review time:** 15 min
**Difficulty:** Medium

---

## 1. The Big Picture

Three short modules finish the preprocessing story. The **normalizer** cleans source text (5 regex passes). The **segmenter** classifies contracts by Solidity era (legacy/transitional/modern). The **parallel pipeline** wraps serial processing in a multiprocessing `Pool` for 8× speedup on 22K-file sources.

---

## 2. Architecture Diagram

```
PreprocessingPipeline._process_one()

  Step 4: normalize(flat.content)          ← Normalizer (39 LOC)
    │   5 regex passes in order:
    │     1. Strip SPDX header
    │     2. Strip /* block comments */
    │     3. Strip // line comments
    │     4. Strip trailing whitespace
    │     5. Collapse 3+ blank lines → 2
    │     + .strip() + '\n'
    │
    └── NormalizeResult(n_lines_before, n_lines_after)

  Step 5: segment_and_bucket(content, pragma_raw)  ← Segmenter (68 LOC)
    │
    ├── Extract contract names (contract Foo {, Foo is Bar {)
    ├── _bucket(pragma_raw):
    │     major<6 → "legacy" | major<8 → "transitional" | else → "modern"
    │
    └── Detect unchecked {} blocks

Parallel mode (preprocess.py: n_workers > 1):
    │
    └── run_preprocess_parallel(pipeline, sol_files, raw_base, n_workers)
          │
          └── mp.Pool(workers).imap(_process_one_worker, args_iter, chunksize)
                │
                ├── Each worker: fresh PreprocessingPipeline (own dedup state)
                ├── Parent writes meta.json (avoids shared file conflict)
                ├── Best-effort dedup (workers may process same file independently)
                └── Errors captured → dropped.csv ("worker_exception")
```

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| 5-regex pass order and WHY | 12 | ☐ |
| `strip() + '\n'` invariant | 12 | ☐ |
| Block comments before line comments | 12 | ☐ |
| SPDX removal is explicit (not just caught by line comment regex) | 12 | ☐ |
| Version buckets: legacy, transitional, modern | 13 | ☐ |
| `_CONTRACT_RE` misses `abstract contract`, `interface`, `library` | 13 | ☐ |
| Multi-contract files kept as single unit | 13 | ☐ |
| Pre-0.8 → `in_unchecked=0.5` era proxy | 13 | ☐ |
| Module-level picklable workers | 14 | ☐ |
| Fresh pipeline per worker (stateful dedup) | 14 | ☐ |
| Parent writes meta.json (not worker) | 14 | ☐ |
| Best-effort dedup in parallel | 14 | ☐ |
| Chunksize = total // (workers × 16) | 14 | ☐ |
| `pool.imap` vs `pool.map` (lazy results) | 14 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: Why does block comment removal come before line comment removal in the normalizer?**

A line like `/* some text // still in a comment */` — if line comments ran first, the `//` inside the block comment would trigger removal, producing `/* some text ` (unclosed). Running block comments first removes the entire `/* ... */` block in one pass.

**Q2: The segmenter keeps multi-contract files as a single unit. Why not split per contract?**

Splitting would break inheritance chains: `contract A { ... } contract B is A { ... }` — separating A and B into different files would make `B is A` unresolvable. Instead, `contract_names` captures all names and downstream stages match labels within the file.

**Q3: In parallel mode, why does the parent write meta.json instead of the worker?**

Avoids shared file write conflicts. If two workers both wrote to the same out_path (possible for duplicate files processed by different workers), one would overwrite the other's partial write. The parent writes meta.json AFTER receiving the worker's result, guaranteeing serialized writes.

**Q4: What is the practical impact of best-effort dedup in parallel mode?**

Two workers might both process the same file before either registers the SHA. Both write `<sha256>.sol` — since filenames are content-addressed, the last write wins trivially (identical content). The dedup "miss" just means slightly inflated output counts. The splitting stage catches cross-file duplicates later.

---

## 5. Session-by-Session Memory Triggers

- **S12:** `_SPDX_RE`, `_BLOCK_CMT`, `_LINE_CMT`, `_TRAIL_WS`, `_MULTI_NL`, `strip() + '\n'`
- **S13:** `_CONTRACT_RE`, `_PRAGMA_VER`, `_UNCHECKED_RE`, `_bucket()`, multi-contract policy
- **S14:** `_process_one_worker`, fresh pipeline, parent writes meta, `pool.imap`, chunksize tuning

---

## 6. What To Remember

1. **Normalizer output has `strip() + '\n'` invariant** — every preprocessed file ends with `\n`.
2. **Multi-contract files stay as one unit** — contract_names list handles disambiguation.
3. **Parallel mode is best-effort dedup** — each worker has its own stateful deduplicator.
4. **Errors captured, not thrown** — a single corrupted file doesn't kill the pool.
5. **`imap` over `map`** for progressive feedback on long runs.
