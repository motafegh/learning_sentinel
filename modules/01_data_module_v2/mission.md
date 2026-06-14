# Mission: `sentinel-data` (Data Module v2)

## Why
The v2 data pipeline is the critical path for Run 11 (launch 2026-08-18).
To confidently debug, modify, and extend it, I need to read every file in
`sentinel_data/` and explain its role in the pipeline, the contracts
between stages, and the design decisions encoded in the code.

## Success looks like
- Read any file in `sentinel_data/` and explain its role + I/O
- Trace data through the full pipeline (ingest → export) and predict
  what artifacts will exist at each stage
- Explain the two-taxonomy divergence (representation order vs labeling
  order) and the byte-identical thin-adapter pattern from memory
- Identify the 2 fixed bugs (EMITS, predictor tier) and the 1 open bug
  (CALL_ENTRY cross-function) by reading the code
- Explain why each deferred item is deferred (DeFiHackLabs, BCCC, the
  12 additive sources, the v3.1 representation stubs)

## Constraints
- **Read-only** on `~/projects/sentinel/` (global P19)
- **Markdown** format for all learning material (global P20)
- **Source code is the only source of truth** — no MEMORY.md facts in
  learning material
- Stage 3 label CLI is a STUB (`cli.py:228-234` prints "NOT IMPLEMENTED"); teach the STUB, not how it "would work"
- 22+ pre-existing test failures in `ml/tests/test_preprocessing.py`
  are out of scope (pre-Stage-7B drift, not blockers)

## Out of scope
- The training loop (Run 10/11) — separate module
- The data_module's predecessor (Phase 1–5 BCCC investigation at
  `docs/legacy/bccc_deep_dive/`) — referenced as historical context,
  not deep-dived
- The 17 source corpora in detail — learned as config entries, not
  data engineering artifacts
- The ML-side training code (`ml/src/preprocessing/`, `ml/src/models/`,
  etc.) — separate modules
