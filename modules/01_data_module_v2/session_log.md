# Session Log — `sentinel-data` Module

> Per-session progress record. Append-only.
> Format per global `reference.md` spec file update protocol.

---

## Format

```
## Session N — <chunk name>

**File:** path (lines)
**Concepts taught:** bullet list
**Warm-up recall:** pass/fail per question, gaps noted
**Challenge questions:** answered Y/N, gaps closed
**Audit flags raised:** A# list
**3 things to lock in:** bullet list (if applicable)
```

---

## Sessions

### Session 01 — Orientation

**File:** `README.md`, `config.yaml`, `docs/architecture.md`, `dvc.yaml`, `pyproject.toml`, `__init__.py`, `cli.py:1-120`
**Status:** 🟡 In progress — questions pending (Q1–Q4)
**Concepts taught:**
- Two `class_names()` orders (representation vs labeling) and the 3 swap points (indices 5, 8, 9)
- 9-stage pipeline flow + per-stage contracts (input/output/sidecar)
- Three-layer config (config.yaml / dvc.yaml / pyproject.toml) — each with a distinct role
- v9 schema constants (LOCKED): NODE_FEATURE_DIM=12, NUM_NODE_TYPES=14, NUM_EDGE_TYPES=12, NUM_CLASSES=10
- One-way dependency rule and why it matters for testability/blast radius
- Stage 7 (export) is a STUB; Stage 3 label CLI is a STUB

**Warm-up recall:** N/A (first session)
**Challenge questions:** 4 posted (Q1–Q4), answers pending
**Audit flags raised:** none
**3 things to lock in:**
1. Two `class_names()` orders diverge at indices 5, 8, 9; index-aligned work uses representation order
2. v9 schema constants are LOCKED; changing them requires bumping FEATURE_SCHEMA_VERSION + updating 3+ files
3. Stage 7 export is a STUB — don't speculate on how it "would work"

**Session doc:** [`sessions/01_orientation.md`](./sessions/01_orientation.md)

---

### Session 02 — CLI Entry Point (`__init__.py` + `cli.py:1-200`)

**File:** `__init__.py` (57 LOC), `cli.py:1-200` (sys.path bootstrap + STAGES + dispatch data + helpers + first handler template)
**Status:** 🟡 In progress — questions pending (Q1–Q4)
**Concepts taught:**
- Package docstring is the architecture in prose (one-way dependency, reproducibility contract, thin-adapter contract)
- CLI module docstring: why a CLI (ops, config centralization, DVC integration)
- `sys.path` bootstrap at `cli.py:62-68` — the ONLY place in production code that mutates `sys.path`
- `STAGES` + `STAGE_DESCRIPTIONS` at module scope — single source of truth for which stages exist
- Lazy imports for `yaml` and per-stage functions (faster cold start, optional deps stay optional)
- `_run_*` handler template: lazy import → load config → resolve data_dir → print header → branch on args

**Warm-up recall:** N/A (first code session)
**Challenge questions:** 4 posted (Q1–Q4), answers pending
**Audit flags raised:** none
**3 things to lock in:**
1. `sys.path` bootstrap at `cli.py:62-68` is the only place `sys.path` is mutated in production; removing it breaks thin-adapter imports from any CWD
2. `STAGES` (line 71) is the single source of truth — `_STAGE_FN`, `dvc.yaml`, `README.md` must all follow
3. The `_run_*` handler template is uniform across all 9 stages; what varies is per-stage args and per-stage Python entry points

**Session doc:** [`sessions/02_cli_entry_point.md`](./sessions/02_cli_entry_point.md)

---

### Session 03 — CLI Handlers + Dispatch (`cli.py:200-700`)

**File:** `cli.py:200-700` (all 9 per-stage handlers + `_resolve_corpus_paths` helper)
**Status:** 🟡 In progress — questions pending (Q1–Q4)
**Concepts taught:**
- `_run_preprocess`: `--workers`, `--sample`, `--retry-failed` args
- `_run_represent`: cache eviction via `check_and_evict` + `write_registry` — the Run 8 v8/v9 silent-mix fix
- `_run_label`: CLI STUB (7 lines, prints "NOT IMPLEMENTED")
- `_run_split`: Contract object builder from `.labels.json` + split triumvirate (stratified_split → dedup_enforcer → nonvulnerable_cap)
- `_run_register`: Catalog SQLite + artifact hash + DatasetVersion
- `_run_verify`: 5 sub-checkers orchestrated; only stage returning int exit code (`--strict`)
- `_run_analyze`: 5 analysis tools with `--only` filter; `_resolve_corpus_paths` helper
- `_run_export`: ready/skipped sources filter; `chunk_export` call (underlying module is STUB)
- `_run_freshness`: 4-line thin wrapper utility
- 3-layer defense pattern in orchestrators (verify + analyze)

**Warm-up recall:** Session 02 concepts (sys.path bootstrap, STAGES, handler template)
**Challenge questions:** 4 posted (Q1–Q4), answers pending
**Audit flags raised:** none
**3 things to lock in:**
1. `_run_verify` is the only stage returning an int exit code; `--strict` turns gate FAIL into exit code 1
2. Cache-eviction fix in `_run_represent` (lines 202-204 + 225) is the operational reason Run 9 v11 results are trustworthy
3. `_run_split` tier-inference heuristic: takes the first positive class's tier; multi-label contracts with mixed tiers get the highest-confidence tier

**Session doc:** [`sessions/03_cli_handlers_and_dispatch.md`](./sessions/03_cli_handlers_and_dispatch.md)

---

### Session 04 — CLI Orchestrator (`cli.py:700-1053`)

**File:** `cli.py:700-1053` (`_STAGE_FN` dispatch dict + `_build_parser` ~220 LOC + `_handle_run` pipeline runner + `main()` entry point)
**Status:** 🟡 In progress — questions pending (Q1–Q4)
**Concepts taught:**
- `_STAGE_FN` dispatch dict vs `STAGES` list — different purposes, both needed
- `_build_parser`: 11 subparsers (1 `run` + 9 stages + 1 `freshness`); conditional args per stage via `if stage == ...` blocks
- `_handle_run`: walks `STAGES[start_idx:]` with synthetic default Namespace; `run` mode can't customize per-stage args
- `main()`: 3 branches (no-command → help, `run` → orchestrator, else → single-stage dispatch)
- Verify is the only stage returning int exit code; `isinstance(result, int)` propagates it to `sys.exit`

**Warm-up recall:** Session 03 concepts (handler template, cache eviction, tier inference)
**Challenge questions:** 4 posted (Q1–Q4), answers pending
**Audit flags raised:** none
**3 things to lock in:**
1. `_STAGE_FN` = CLI commands; `STAGES` = pipeline DAG stages. Adding a stage requires updating both.
2. `_handle_run` constructs default Namespace — `run` mode uses defaults, per-stage mode allows customization.
3. `main()` has 3 branches: no-command (help), `run` (orchestrator), else (single-stage dispatch via `_STAGE_FN`).

**Session doc:** [`sessions/04_cli_orchestrator.md`](./sessions/04_cli_orchestrator.md)

---

### Session 05 — Ingestion Overview (`ingestion/ingest.py` + `manifest.py` + `freshness.py`)

**File:** `ingestion/ingest.py` (113 LOC) + `manifest.py` (111 LOC) + `freshness.py` (120 LOC)
**Status:** ✅ Completed (answers filled)
**Concepts taught:**
- `_all_sources` merges 3 config sections (critical_path → additive → legacy) in override order
- `_source_config` converts raw config dict to typed `SourceConfig` dataclass; `extra` dict catches source-specific keys
- `ingest_source` 5-step call chain: resolve source → build SourceConfig → resolve dest → get connector → pull → build+save manifest
- `IngestionManifest` is append-only; re-ingest re-validates every SHA-256 (MISSING vs CHANGED)
- Freshness check is informational, not blocking; flags stale git pins and Slither version mismatches
- Strategy pattern preview: `get_connector()` returns `GitConnector`/`ManualConnector`/etc.
- Audit flag A#001: manifest verified at preprocessing but not at representation

**Warm-up recall:** Session 04 concepts (dispatch dict, `run` mode default Namespace, 3-branch `main()`)
**Challenge questions:** 4 posted (Q1–Q4), answers filled in §0.6
**Audit flags raised:** A#001 (manifest not verified at representation entry)
**3 things to lock in:**
1. Manifest is append-only with SHA-256 per file — the audit trail for "what version did we train on?"
2. `_all_sources` merges 3 config sections; later overrides earlier (legacy can patch v2)
3. Freshness check is informational — Slither check exists because API changes broke Run 9

**Session doc:** [`sessions/05_ingestion_overview.md`](./sessions/05_ingestion_overview.md)

---

### Session 06 — Ingestion Connectors (`connectors/base.py` + `git_connector.py` + `manual_connector.py`)

**File:** `connectors/base.py` (95 LOC) + `__init__.py` (38 LOC) + `git_connector.py` (89 LOC) + `manual_connector.py` (189 LOC)
**Status:** 🟡 In progress — questions pending (Q1–Q4)
**Concepts taught:**
- Strategy pattern: `BaseConnector` ABC, `_pull()` template method, `find_sol_files()` shared static method
- Registry pattern: `_REGISTRY` dict (7 entries → 5 unique connectors) + `get_connector()` factory
- Git clone strategy: full clone (pinned) vs shallow clone (unpinned); idempotent re-runs skip cloning
- Manual materialization: 3 cases (zip → extract + symlink/copy, dir → symlink/copy, glob → resolve then handle)
- `post_clone_cmd` via `cfg.extra` for scabench's `python checkout_sources.py`
- macOS `__MACOSX/` stripping in zips (DIVE: 44k zip entries, 22k .sol, 22k macOS noise)
- Zip-slip defense in `_extract_zip`

**Warm-up recall:** Session 05 concepts (extra dict flow, manifest contract, pin vs resolved_pin)
**Challenge questions:** 4 posted (Q1–Q4), answers pending
**Audit flags raised:** none
**3 things to lock in:**
1. `find_sol_files` is shared by all connectors; `include_subdirs` limits search space, `exclude_subdirs` filters results
2. Git connector: pinned → full clone (history needed for checkout), unpinned → shallow `--depth 1` (speed)
3. Manual connector "symlink" mode for zips = extract to `__zip_extracted/` + symlink; preserves option to delete symlink without re-extracting

**Session doc:** [`sessions/06_ingestion_connectors.md`](./sessions/06_ingestion_connectors.md)
