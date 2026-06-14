# Audit Flags — `sentinel-data` Module

> All `[AUDIT]` flags raised while learning `sentinel-data`. Append-only.
> Format per global `reference.md` spec file update protocol.

---

## Format

```
## A# — <file> — <short description>

**File:** path
**Location:** function/line
**Issue:** what is wrong and why it matters
**Fix:** concrete fix
**Severity:** Low / Medium / High
**Status:** Open / Noted / Fixed
**Raised:** Session N, Chunk N
```

---

## Flags

## A#001 — `manifest.py:verify` — Manifest verified at preprocessing but not at representation

**File:** `sentinel_data/ingestion/manifest.py` + `sentinel_data/preprocessing/preprocess.py`
**Location:** `manifest.py:63-77` (verify), `preprocess.py` (calls at load time)
**Issue:** The manifest is verified when preprocessing loads a source, but NOT at
representation time. If a `.sol` or `.meta.json` file is corrupted or changed between
preprocessing and representation, the `.pt` graph will be built from corrupt data without
detection. A `verify_manifest()` call at representation entry would catch this.
**Fix:** Add a manifest verification call at the start of `represent_source()` in
`sentinel_data/representation/orchestrator.py`.
**Severity:** Low (in-practice: representation runs immediately after preprocessing
in normal pipeline order; corruption between stages is unlikely in production)
**Status:** Noted
**Raised:** Session 05, Chunk 05

---

## A#002 — `data_module/README.md` — Contradictory export stage status

**File:** `data_module/README.md` (line 4) vs `sentinel_data/README.md` (line 3)
**Location:** `data_module/README.md:4` says export is "⏳ STUB"; `sentinel_data/README.md:3` says export is "✅ COMPLETE"
**Issue:** The two READMEs contradict each other. The code shows export is fully
implemented (7 modules, 1482 LOC): `chunk_export()` orchestrates 4 writers,
computes artifact hash, and writes manifest.json last (Fix A). `SentinelDatasetExport`
is a full consumer-facing API. The parent `data_module/README.md` is stale.
**Fix:** Update `data_module/README.md:4` to match `sentinel_data/README.md:3` (export ✅ COMPLETE).
**Severity:** Low (cosmetic — doesn't affect code correctness)
**Status:** Open
**Raised:** Session 06 audit, Chunk 06
