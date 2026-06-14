# Session 03 — CLI Handlers + Dispatch (`cli.py:200-700`)

**Date:** 2026-06-13
**Session number:** 03 of 36
**Mode:** Understand (the template from Session 02 covers most of this)
**Estimated study time:** 1 hr teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0 Recap of Session 02 + Model Answers to Q1–Q4

### §0.1 Recap of Session 02

Session 02 read `__init__.py` (57 LOC) + `cli.py:1-200`. Key
concepts: **(1)** the package docstring is the architecture in prose
(one-way dependency rule, reproducibility contract, thin-adapter
contract); **(2)** the CLI module docstring explains the design
choices for using a CLI (ops, config centralization, DVC integration);
**(3)** the `sys.path` bootstrap at lines 62-68 is the **only** place
in production code that mutates `sys.path`; **(4)** `STAGES` +
`STAGE_DESCRIPTIONS` are the dispatch data, at module scope so other
modules can import them; **(5)** the `_run_*` handler template is
5 steps: lazy import → load config → resolve `data_dir` → print
header → branch on args → dispatch.

### §0.2 Model answer to Q1 [Mechanism]

> **Q:** What breaks if the `sys.path` bootstrap at `cli.py:62-68`
> is removed? List 3 concrete failure modes.

**Model answer:**

1. **Docker container with a different CWD** — `sentinel-data
   ingest` fails with `ModuleNotFoundError: No module named 'ml'`
   because `ml/src/preprocessing/graph_schema.py` can't be
   imported. The thin-adapter `representation/graph_schema.py`
   does `from ml.src.preprocessing.graph_schema import …` at
   module load time, and `ml/` is not on `sys.path` from inside
   the container's CWD.
2. **CI runner with a clean checkout** — same error. The runner
   checks out the repo, `cd`s into `data_module/`, runs `pytest`.
   The test `conftest.py` does the same `sys.path` trick (per
   the docstring at `cli.py:47-49`), so tests pass; but if the
   test exercises the CLI via `subprocess.run(["sentinel-data",
   "ingest"])`, the subprocess inherits the CWD but the `sys.path`
   manipulation in `conftest.py` is gone, and the import fails.
3. **`sentinel-data` as a standalone PyPI package** — impossible.
   Any user who installs `sentinel-data` via `pip install
   sentinel-data` (without `ml/`) gets a broken package: the
   thin-adapter imports fail on first use. The bootstrap in
   `cli.py:62-68` is what makes the package "look standalone"
   while still depending on `ml/`. Without it, even the simple
   `from sentinel_data import __version__` would work, but the
   first call to a stage that touches a thin-adapter would crash.

**Common misconception:** "We could just `pip install -e ./ml` and
add it to dependencies." The `pyproject.toml` is a **single
package** (`sentinel-data`). Adding a dep on `sentinel-ml` would
violate the one-way dependency rule (P19 + module P103). The
bootstrap is the **only** way to satisfy both "no sentinel-ml dep
in pyproject" and "thin-adapter can import from `ml/src/`."

### §0.3 Model answer to Q2 [Pattern]

> **Q:** The `_run_ingest` handler uses `getattr(args, "source",
> None)` instead of `args.source`. Why is the defensive `getattr`
> form used, and what would happen if the per-stage parser forgot
> to add `--source` for a stage that doesn't support per-source
> runs?

**Model answer:**

`argparse.Namespace` is a simple attribute holder. Accessing an
attribute that doesn't exist raises `AttributeError`. The
defensive `getattr(args, "source", None)` form returns `None` if
the attribute is missing, and the actual value if it exists.

**Why this matters here:** the per-stage subparsers in
`_build_parser` add `--source` **only when the stage supports
per-source runs**. For example, the `register` subparser doesn't
add `--source` (you register a whole dataset, not one source at
a time); the `analyze` subparser also doesn't. But the
`_run_register` and `_run_analyze` handler code still wants to
check `args.source` for the same all-vs-specific branching that
`_run_ingest` uses. If they used `args.source`, every `register`
or `analyze` invocation would crash with `AttributeError`; with
`getattr(args, "source", None)`, they get `None` and fall through
to the "all" branch.

**What would happen if the per-stage parser forgot to add
`--source` for a stage that doesn't support per-source runs** (and
the handler used `args.source` instead of `getattr`): the user
would see `AttributeError: 'Namespace' object has no attribute
'source'` on every invocation of that stage, even though the stage
logically has no per-source mode. The defensive form prevents this
class of bug entirely.

**Common misconception:** "I should always use `args.X` because
it's cleaner." The trade-off: `args.X` is cleaner when the parser
guarantees the attribute exists; `getattr(args, "X", default)` is
safer when the parser is conditional. In the CLI, where the parser
is built dynamically per stage, the defensive form is the right
default.

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** The CLI uses **lazy imports** for `yaml` (in `_load_config`)
> and for per-stage functions (e.g. `from sentinel_data.ingestion.ingest
> import ingest_all`). What's the general principle, and what are
> the trade-offs vs. eager top-of-file imports?

**Model answer:**

**Principle:** defer expensive imports until they're actually needed.
Three benefits:
1. **Faster cold start** — fewer modules loaded on `import cli`.
2. **Smaller import graph** — if the user only ever calls
   `sentinel-data freshness` (which doesn't load `yaml`), the
   `yaml` module is never imported.
3. **Optional dependencies** — the per-stage functions can depend
   on heavy packages (e.g. `slither-analyzer`) that aren't needed
   unless that stage is called. A user who only runs `ingest` +
   `preprocess` doesn't need Slither installed.

**Trade-offs vs. eager imports:**

| Pro (lazy) | Con (lazy) |
|------------|-----------|
| Faster cold start | Errors surface at call time, not import time (less stack context) |
| Smaller import graph | First call has the import overhead |
| Optional deps stay optional | Can hide circular imports that eager imports would catch at startup |
| Easier to add new stages | IDE autocomplete less reliable (the symbol isn't "real" until import runs) |

**Where else this pattern appears:**
- **Plugin systems** (e.g. `pluggy` in pytest, `stevedore` in
  OpenStack) — plugins are loaded on demand.
- **Web servers** (Django, Flask) — views are imported lazily
  in some configurations.
- **`__getattr__` proxies** (e.g. `tensorflow.compat.v1`,
  `torch._C`) — submodule access triggers the import.
- **Jupyter notebooks** — `%autoreload` is a form of lazy
  re-import.
- **Python's own import system** — `__import__('module')` is
  lazy until the module is actually accessed.

**Common misconception:** "Lazy imports are always better." They're
better for **optional or expensive** dependencies. For core
dependencies used in every code path, eager imports catch missing
dependencies at startup (with a clear error) rather than at the
first call (potentially after the user has waited minutes for
something else to complete).

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** If you add a 10th stage (e.g. `drift_report`) to the CLI,
> list every place in `cli.py` that you must update for the new
> stage to be callable from `sentinel-data <stage>`.

**Model answer (8 places in `cli.py` + 2 outside):**

Inside `cli.py`:

1. **`STAGES` list** (line 71-81) — add `"drift_report"` at the
   correct position in the pipeline order.
2. **`STAGE_DESCRIPTIONS` dict** (line 83-93) — add a one-line
   description following the pattern of the others.
3. **Define a `_run_drift_report` function** (~30-50 LOC) following
   the 5-step template: lazy import → load config → resolve
   `data_dir` → print header → branch on args → dispatch.
4. **`_STAGE_FN` dispatch table** (line 745-756) — add
   `"drift_report": _run_drift_report,`.
5. **`_build_parser`** (line 761+) — add a new `subparsers.add_parser("drift_report", ...)`
   block (could be 30-100 LOC depending on the stage's args).
6. **`_handle_run`** (line 980-1011) — the `argparse.Namespace`
   defaults list at lines 996-1003 may need new fields for the
   new stage's args (e.g. `--baseline-version` for `drift_report`
   is already there, but a hypothetical `--window-size` would
   need to be added with a default).
7. **Module docstring** (line 23-33) — add `drift_report` to the
   stages list inside the docstring's "Stages (in pipeline order)"
   section.
8. **`__all__`** — if `cli.py` defines an `__all__` (it doesn't
   currently), add the new function name. Not needed here, but a
   common gotcha in other files.

Outside `cli.py`:

9. **`dvc.yaml`** — add a new `stages: drift_report:` block with
   `cmd`, `deps`, `outs`.
10. **`README.md`** (and possibly `00_INDEX.md`) — update the
    stage status table and the pipeline DAG.

**Common misconception:** "I just need to add it to `_STAGE_FN`."
That alone will cause `argparse` to reject `sentinel-data
drift_report` with "unknown command" because the subparser isn't
registered. The dispatch table is the **last** thing that needs
to be updated, not the first. **The order is: data structures →
handler function → dispatch table → subparser → orchestrator
defaults → docs.**

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** _pending_

**Your answer to Q2:** _pending_

**Your answer to Q3:** _pending_

**Your answer to Q4:** _pending_

---

## §1 What This Chunk Covers (per global P5)

`cli.py:200-700` — the **per-stage handler bodies** for all 9
stages + the orchestrator helper (`_resolve_corpus_paths`). The
template from Session 02's §7 applies to every handler; this
session highlights what's **unique** to each.

| Handler | Lines | What's unique |
|---------|-------|---------------|
| `_run_preprocess` | 131-157 | `--workers`, `--sample`, `--retry-failed` |
| `_run_represent` | 160-225 | cache eviction via `check_and_evict`; writes version registry |
| `_run_label` | 228-234 | **STUB** — prints "NOT IMPLEMENTED" |
| `_run_split` | 237-314 | builds `Contract` objects from `.labels.json`; runs `stratified_split` + `dedup_enforcer` + `nonvulnerable_cap`; writes splits + manifest |
| `_run_register` | 317-383 | opens `Catalog` SQLite; computes artifact hash; registers `DatasetVersion` |
| `_run_verify` | 386-486 | orchestrates 5 sub-checkers in sequence; calls gate; **only stage that returns an int exit code** |
| `_resolve_corpus_paths` | 489-521 | helper for `--corpus` arg (current build vs registered version) |
| `_run_analyze` | 524-663 | orchestrates 5 analysis tools; supports `--only` and `--baseline-version` |
| `_run_export` | 666-733 | calls `chunk_export`; prints manifest summary |
| `_run_freshness` | 736-742 | thin wrapper around `run_freshness_check` |

---

## §2 `_run_preprocess` (131-157) — The Workers/Sample/Retry Triumvirate

```python
# Learning mode: Understand | Same template, three new args
def _run_preprocess(args: argparse.Namespace) -> None:
    from sentinel_data.preprocessing.preprocess import preprocess_all, preprocess_source
    cfg = _load_config(args.config)
    data_dir = Path(args.config).parent / "data"

    print(f"[preprocess] {STAGE_DESCRIPTIONS['preprocess']}")
    print(f"  config : {args.config}")
    n_workers = getattr(args, "workers", 1)
    sample = getattr(args, "sample", None)
    retry_failed = getattr(args, "retry_failed", False)
    if n_workers > 1:
        print(f"  workers : {n_workers}")
    if sample:
        print(f"  sample  : {sample} files (--sample)")
    if retry_failed:
        print(f"  mode    : retry-failed (re-run only files in dropped.csv)")
    # ... dispatch to preprocess_all or preprocess_source
```

**3 args unique to preprocess:**

- **`--workers N`** (default 1) — parallel preprocessing via
  `multiprocessing`. The handler passes `n_workers` to
  `preprocess_all` / `preprocess_source`. Implementation in
  `preprocessing/parallel.py` (covered in Session 11).
- **`--sample N`** (default None = all files) — process only the
  first N files. Useful for smoke testing and debugging the
  preprocessing pipeline on a tiny subset.
- **`--retry-failed`** (default False) — re-run only the files
  in `dropped.csv` (from a previous failed run). Avoids
  re-processing the 99% that succeeded.

**Why the conditional `print` blocks (lines 141-146):** to keep
the dry-run output minimal. If the user didn't pass `--workers
2`, don't print "workers: 1" — it's the default. The same logic
applies to `--sample` and `--retry-failed`. This is good CLI
hygiene: don't print defaults.

---

## §3 `_run_represent` (160-225) — Cache Eviction + Version Registry

```python
# Learning mode: Master🔵 | The Run-8 v8/v9 silent-mix fix lives here
def _run_represent(args: argparse.Namespace) -> None:
    from sentinel_data.representation.orchestrator import represent_source
    from sentinel_data.representation.versioner import check_and_evict, write_registry, current_versions
    # ... (resolve sources from config if --source not given)
    schema_v, extractor_v = current_versions()  # ← Master: read live versions
    # ... (per-source loop)
    for source in sources:
        evicted = check_and_evict(representations_root, source, schema_v, extractor_v)
        if evicted:
            print(f"    evicted {evicted} stale cache entries")
        result = represent_source(source, cfg, data_dir, limit=limit, force=force, emit_cfg=emit_cfg)
        # ... (print stats)
    write_registry(representations_root, schema_v, extractor_v)  # ← Master: stamp the registry
```

**The Run-8 silent-mix fix** (per `representation/README.md:135`):

In Run 8, the v8 schema graphs and the v9 schema graphs ended up
mixed in the cache directory. The model was trained on a mix of
both, producing nonsense results. The diagnosis took days. The
fix is here in `_run_represent`:

1. **`check_and_evict`** (called per source) compares the
   registry's `(_version_registry.json`) to the live
   `(schema_version, extractor_version)`. On mismatch, it
   evicts the stale cache entries for that source.
2. **`write_registry`** (called after the per-source loop)
   stamps the current `(schema_v, extractor_v)` into
   `_version_registry.json`. Next run will read this and
   detect a mismatch if either version changed.

**The `current_versions()` helper** is a thin re-export from
`representation/versioner.py:100` (covered in Session 14).

**`--limit N`** caps contracts per source; **`--force`** bypasses
the cache; **`--emit-cfg`** also writes standalone CFG artifacts
(used by Stage 6 `complexity_proxy_risk`).

**The `RepresentResult` dataclass** (lines 214-223 prints 8
counters) is defined in `representation/orchestrator.py:249-264`.
We cover its fields in Session 13.

---

## §4 `_run_label` (228-234) — The CLI STUB

```python
# Learning mode: Awareness | The merger runs from Python today (per README:170)
def _run_label(args: argparse.Namespace) -> None:
    print(f"[label] {STAGE_DESCRIPTIONS['label']}")
    print(f"  config : {args.config}")
    if args.dry_run:
        print("  (dry-run — no files written)")
        return
    print("  NOT IMPLEMENTED — implement in Stage 3")
```

**Why this is a STUB** (per `README.md:170`):

> "Stage 3 — Label ... `sentinel-data label [--source NAME]`. Apply
> per-source crosswalk YAMLs + merge + 99% co-occurrence flag +
> Go/No-Go gate. ⚠️ CLI STUB (`cli.py:223-229`); merger runs from
> Python today."

The merger (`labeling/merger.py`, 240 LOC) is fully implemented
and called from `data_module/tests/test_labeling/test_merger.py`
and from notebook-style scripts. The CLI handler is a 7-line
placeholder that prints a "NOT IMPLEMENTED" message.

**Implication for Run 10/11:** the v2 build uses the merger via
Python imports (e.g. `from sentinel_data.labeling.merger import
merge_labels`). The CLI subcommand is a convenience that's not yet
wired. Per module P104, treat the STUB as-is — don't speculate on
how the implementation "would work."

---

## §5 `_run_split` (237-314) — The Contract Builder + Splitter Triumvirate

```python
# Learning mode: Understand | Builds Contract objects from .labels.json then dispatches
def _run_split(args: argparse.Namespace) -> None:
    from sentinel_data.splitting import (
        Contract, apply_dedup_enforcer, apply_nonvulnerable_cap,
        apply_strategy, stratified_split, write_manifest, write_splits,
    )
    # ... (load config, resolve data_dir)

    # Load graph-hash dedup groups if available (fixes Level-3 stub)
    dedup_groups_path = data_dir / "dedup_groups_graph_hash.json"
    cid_to_group: dict[str, str] = {}
    if dedup_groups_path.exists():
        dg_data = json.loads(dedup_groups_path.read_text())
        cid_to_group = dg_data.get("groups", {})
        print(f"  Loaded {len(cid_to_group)} dedup groups "
              f"({dg_data.get('n_unique_groups',0)} unique)")
    else:
        print(f"  WARNING: dedup_groups not found; dedup_enforcer will be ineffective")

    # Read merged labels and build Contract objects
    for p in sorted(merged_dir.glob("*.labels.json")):
        lj = json.loads(p.read_text())
        sha = lj["sha256"]; source = (lj.get("sources") or ["unknown"])[0]
        tier = "T0"
        for cls, entry in lj.get("classes", {}).items():
            if entry.get("value") == 1: tier = entry.get("tier") or "T0"; break
        classes = {cls: entry.get("value", 0) for cls, entry in lj.get("classes", {}).items()}
        primary = next((c for c, e in lj.get("classes", {}).items() if e.get("value") == 1), "NonVulnerable")
        n_pos = sum(1 for e in lj.get("classes", {}).items() if e.get("value") == 1)
        contracts.append(Contract(sha256=sha, source=source, tier=tier, classes=classes, primary_class=primary, n_pos=n_pos, dedup_group=cid_to_group.get(sha)))
    splits = stratified_split(contracts, seed=args.seed)
    apply_dedup_enforcer(splits)
    apply_nonvulnerable_cap(splits, cap=args.nonvuln_cap, seed=args.seed)
    write_splits(splits, out_dir)
    write_manifest(splits, out_dir)
    # ... (print summary)
```

**The Contract object builder (lines 263-285)** is a key piece of
glue code: it transforms the on-disk `.labels.json` shape (per
`labeling/merger.py`) into the `Contract` dataclass (defined in
`splitting/splitters.py`). Four things to notice:

- **Source fallback:** `(lj.get("sources") or ["unknown"])[0]` —
  if a label file is missing the `sources` field, fall back to
  `"unknown"`. Defensive.
- **Tier inference:** the tier is taken from the **first class
  that is positive** (lines 289-292). If a contract has multiple
  positives from different sources, the highest-confidence tier
  wins. This is a heuristic; for multi-source contracts the
  tier field could be a list, but the splitter only needs one.
- **Primary class:** the first class with `value=1`, or
  `"NonVulnerable"` if none. The `next(... , default)` idiom
  handles the empty case.
- **Graph-hash dedup groups** (lines 264-276): before building
  Contract objects, the handler loads `dedup_groups_graph_hash.json`
  from `data/` — a file written by `sentinel-data compute-dedup-groups`
  after representation. Each SHA-256 maps to a dedup group ID.
  If the file doesn't exist, `cid_to_group` stays empty and a
  WARNING is printed — the `dedup_enforcer` will have no groups
  to act on (all groups are singletons). This is the "Level-3
  stub" fix: without dedup groups, cross-source duplicates pass
  through the splitter undetected.

**The split triumvirate** (lines 290-298):

1. **`stratified_split(contracts, seed=...)`** — runs the
   stratified strategy. The `splitters.py:441` file has 4
   strategies total (random, time, hash, stratified).
2. **`apply_dedup_enforcer(splits)`** — post-split dedup fix
   for the BCCC failure pattern. If the same contract ended up
   in train AND val (because dedup was pre-stratification),
   this removes the val copy. (Covered in Session 29.)
3. **`apply_nonvulnerable_cap(splits, cap=...)`** — caps the
   NonVulnerable class at `3:1` per `pipeline.negative.
   positive_ratio_max` in `config.yaml`.

**The output:**
- `data/splits/v{args.version}/{train,val,test}.jsonl` — the
  actual split files (jsonl = one contract per line).
- `data/splits/v{args.version}/split_manifest.json` —
  metadata: per-class counts, dedup groups resolved,
  NonVulnerable:positive ratio.

The summary at lines 307-314 prints the split sizes + the
`dedup_groups_resolved` count + the NonVulnerable:positive ratio.

---

## §6 `_run_register` (317-383) — Catalog + Artifact Hash

```python
# Learning mode: Understand | Opens SQLite, computes hash, registers DatasetVersion
def _run_register(args: argparse.Namespace) -> None:
    from sentinel_data.registry import (Catalog, DatasetVersion, compute_hash)
    # ... (load config, resolve data_dir)
    # Locate the split manifest
    split_dir = data_dir / "splits" / f"v{args.version}"
    manifest_path = split_dir / "split_manifest.json"
    # ... (error if not found)
    h = compute_hash(manifest_path)
    db_path = data_dir / "registry" / "catalog.db"
    yaml_path = data_dir / "registry" / "catalog.yaml"
    cat = Catalog(db_path, yaml_path)
    version = DatasetVersion(
        name=args.name,
        source_set=args.sources or [],
        split_version=f"v{args.version}",
        preprocessing_config_hash="",  # TODO: hash of config.yaml in Stage 7
        artifact_hash=h,
        artifact_path=str(split_dir),
        verification_report_path=str(data_dir / "verification" / args.verification_report) if args.verification_report else "",
    )
    cat.add_dataset_version(version)
    # ... (optional retire_previous, write YAML mirror)
```

**The artifact hash** is computed by `compute_hash(manifest_path)`.
This is a SHA-256 over the **contents** of the split manifest (and
implicitly over the train/val/test files it references, depending
on the impl). The hash becomes the dataset's identity in the
catalog. Any change to the splits invalidates the hash, which
detects "did the splits drift between register and the next
analyze/export run?"

**`Catalog` is a SQLite + YAML mirror** (per `registry/catalog.py:541`).
The DB has 4 base tables + 2 system tables. The YAML mirror is
for human inspection (you can `cat catalog.yaml` to see the
registered versions).

**`DatasetVersion`** is the dataclass for a registered version.
Fields: name, source_set, split_version, preprocessing_config_hash,
artifact_hash, artifact_path, verification_report_path. The
`preprocessing_config_hash` is **TODO** (line 357) — Stage 7 will
hash the `config.yaml` so version→config traceability is complete.

**`--retire-previous NAME`** (lines 367-373) — soft-deletes a
previous version, marking it as `superseded_by=NAME`. The
catalog keeps the row for history.

---

## §7 `_run_verify` (386-486) — The Big Orchestrator

```python
# Learning mode: Understand | Orchestrates 5 sub-checkers + gate + report
def _run_verify(args: argparse.Namespace) -> None:
    from sentinel_data.verification.class_auditor import run_audit
    from sentinel_data.verification.semantic_checker import run_semantic_check
    # ... (4 more imports)
    # ... (load config, resolve data_dir, print header)
    if args.dry_run: return

    verify_cfg = (cfg or {}).get("pipeline", {}).get("verification", {})
    fp_sample = int(verify_cfg.get("fp_sample_size", 50))
    neg_warn = float(verify_cfg.get("negative_tool_hit_warn", 0.05))
    neg_fail = float(verify_cfg.get("negative_tool_hit_threshold", 0.10))
    skip_tool = bool(args.skip_tool_validator)
    skip_fp = bool(args.skip_fp_estimator)
    skip_neg = bool(args.skip_negative_checker)

    corpus_tag = " + ".join((cfg.get("sources_critical_path") or {}).keys()) or "manual"

    print("\n  [1/5] class_auditor")
    audit = run_audit(data_dir)
    # ... (print stats)

    print("\n  [2/5] semantic_checker")
    sem_limit = int(args.semantic_limit_per_class) if args.semantic_limit_per_class else None
    semantic = run_semantic_check(data_dir, limit_per_class=sem_limit)
    # ...

    tool_validation = None
    if not skip_tool:
        print("\n  [3/5] tool_validator (Slither, may be slow on first run)")
        # ...
    # ... (similar for fp_estimator and negative_checker)

    print("\n  ── Gate ──")
    gate = run_gate(audit, semantic, tool_validation=tool_validation, fp_estimation=fp_estimation, negative_check=negative_check)
    print(gate)

    out_path = data_dir / "verification" / f"verification_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md"
    generate_report(audit, semantic, gate, ..., output_path=out_path)
    print(f"\n  Wrote report: {out_path}")

    if gate.gate_passed: return 0
    if not args.strict:  return 0
    return 1
```

**This is the only stage that returns an int exit code** (line
482-486). Other stages return `None` on success. The
`--strict` flag turns a gate FAIL into exit code 1 (CI-friendly).
Per `README.md:104`:
> "`cli.py:_STAGE_FN` (line 679-690) is the dispatch table. Each
> stage function returns `None` on success, or an int exit code
> (currently only `verify` does this — non-zero on `--strict` mode
> FAIL)."

**The 5 sub-checkers** (covered in Sessions 20-26):
1. **`class_auditor`** — 10×10 co-occurrence matrix, flag at
   P(B|A) > 0.50.
2. **`semantic_checker`** — per-class graph-feature checks; 3/10
   classes are `NOT_EXTRACTABLE`.
3. **`tool_validator`** — Slither per-class agreement.
4. **`fp_estimator`** — stratified-by-(source, tier) FP rate.
5. **`negative_checker`** — Slither hit rate on NonVulnerable
   contracts.

**3 skip flags** (lines 411-413): `--skip-tool-validator`,
`--skip-fp-estimator`, `--skip-negative-checker`. Semantic
checker and class_auditor cannot be skipped (they're fast and
always-on).

**Config-driven thresholds** (lines 408-410): the handler reads
`pipeline.verification.*` from `config.yaml` with sensible
defaults (50 samples/class, 5% warn, 10% fail).

**`corpus_tag`** (line 415) is a human-readable label for the
report header: `"solidifi + dive"` etc. It comes from the
`sources_critical_path` keys.

**The gate** combines all 5 checkers into a per-class verdict
(`VERIFIED` / `PROVISIONAL` / `BEST-EFFORT` / `FAIL`). The report
is a 9-section markdown file with the per-class gate, per-class
stats, co-occurrence matrix, semantic summary, tool validation,
FP estimation, negative checker, hard failures, known limitations.
(Covered in Session 25.)

---

## §8 `_resolve_corpus_paths` (489-521) — The Corpus Resolver Helper

```python
# Learning mode: Understand | Helper for --corpus arg (used by _run_analyze)
def _resolve_corpus_paths(args, data_dir) -> tuple[Path, Path, Path, Path]:
    if not args.corpus:
        return (data_dir / "labels", data_dir / "representations",
                data_dir / "preprocessed", data_dir / "labels" / "merged")
    from sentinel_data.registry import Catalog
    cat = Catalog(data_dir / "registry" / "catalog.db",
                  data_dir / "registry" / "catalog.yaml")
    v = cat.get_dataset_version(args.corpus)
    if v is None:
        print(f"  ERROR: corpus version {args.corpus} not found in catalog")
        return Path(), Path(), Path(), Path()
    artifact = Path(v.artifact_path)
    if not artifact.exists():
        print(f"  ERROR: corpus artifact path does not exist: {artifact}")
        return Path(), Path(), Path(), Path()
    return (artifact / "labels", artifact / "representations",
            artifact / "preprocessed", artifact / "labels" / "merged")
```

**Returns:** `(labels_root, rep_root, preproc_root, merged_dir)`.
4 paths the analysis tools need.

**Two modes:**
1. **`--corpus` not given** → use the current build's `data/`
   subdirs.
2. **`--corpus NAME` given** → look up NAME in the catalog,
   use that registered version's `artifact_path` (which is the
   split dir from `_run_register`).

**The graceful degradation:** if the catalog lookup fails
(unknown name) or the artifact path doesn't exist (e.g. the
user deleted the split dir), return empty `Path()` objects. The
caller (`_run_analyze` line 568-573) checks `if not merged_dir or
not merged_dir.exists():` and prints a clear error.

---

## §9 `_run_analyze` (524-663) — The Analysis Orchestrator

```python
# Learning mode: Understand | Similar to _run_verify but for the 5 analysis tools
def _run_analyze(args: argparse.Namespace) -> None:
    # ... (load config, read thresholds, set up output_dir)
    if args.dry_run: return
    only = args.only
    available = ["balance_viz", "feature_dist", "cooccurrence", "overlap_detector", "drift_monitor"]
    if only and only not in available:
        print(f"  ERROR: --only must be one of {available}")
        return
    output_dir.mkdir(parents=True, exist_ok=True)
    labels_root, rep_root, preproc_root, merged_dir = _resolve_corpus_paths(args, data_dir)
    if not merged_dir or not merged_dir.exists(): ...

    if not only or only == "balance_viz":
        from sentinel_data.analysis.balance_viz import run_balance_viz
        summary = run_balance_viz(merged_dir, output_dir)
        # ...
    if not only or only == "feature_dist":
        from sentinel_data.analysis.feature_dist import run_feature_dist
        summary = run_feature_dist(merged_dir, rep_root, preproc_root, output_dir, sigma_threshold=sigma_threshold)
        # ... (prints high_risk_pairs if any)
    if not only or only == "cooccurrence":
        from sentinel_data.analysis.cooccurrence import run_cooccurrence
        # ...
    if not only or only == "overlap_detector":
        from sentinel_data.analysis.overlap_detector import run_overlap_detector
        # ...
    if not only or only == "drift_monitor":
        from sentinel_data.analysis.drift_monitor import run_drift_monitor
        if args.baseline_version:
            # Look up baseline from catalog
            # ...
        else:
            print("    skipped (no --baseline-version given)")
```

**5 analysis tools** (covered in Sessions 32-33):
- `balance_viz` (per-class/source/tier counts)
- `feature_dist` (the headline — Run-9-failure catcher)
- `cooccurrence` (directed + conditional matrices)
- `overlap_detector` (pairwise source Jaccard)
- `drift_monitor` (KS test for features + labels)

**`--only TOOL`** runs just one tool; the default runs all 5.

**`--corpus NAME`** delegates to `_resolve_corpus_paths` to pick
the current build OR a registered version.

**`--baseline-version NAME`** is for `drift_monitor` specifically:
it compares the current corpus against a previously-registered
version (e.g. `v1.4-bccc`).

**`--run-id ID`** defaults to `datetime.now().strftime("%Y%m%d_%H%M%S")`.
The output dir is `data/analysis/<run_id>/` — keeps separate
analysis runs from clobbering each other.

---

## §10 `_run_export` (682-749) — The Export Handler

```python
# Learning mode: Understand | Calls chunk_export; fully implemented (7 modules, 1482 LOC)
def _run_export(args: argparse.Namespace) -> None:
    from sentinel_data.export import chunk_export
    # ... (load config, resolve split_version + shard_size + output_dir)
    # Resolve enabled sources
    all_sources = [k for k, v in (cfg.get("sources_critical_path") or {}).items()
                   if isinstance(v, dict) and v.get("enabled", False)]
    ready_sources, skipped_sources = [], []
    for s in all_sources:
        if (data_dir / "preprocessed" / s).exists():
            ready_sources.append(s)
        else:
            skipped_sources.append({"name": s, "reason": "preprocessed dir not found"})
    # ... (error if no ready sources, print ready + skipped)
    if args.dry_run: return
    splits_dir = data_dir / "splits" / f"v{split_version}"
    rep_root = data_dir / "representations"
    preproc_root = data_dir / "preprocessed"
    manifest = chunk_export(rep_root=rep_root, preproc_root=preproc_root,
                            splits_dir=splits_dir, output_dir=output_dir,
                            config_path=config_path, shard_size=shard_size,
                            source_set=ready_sources, skipped_sources=skipped_sources)
    # ... (print manifest summary)
```

**The 4 unique args:**
- **`--split-version N`** (default 1) — which split dir to export.
- **`--output-dir PATH`** (default `data/exports/<dataset_version>/`).
- **`--shard-size N`** (default from `config.yaml:export_shard_size`, or 5000).
- **`--dataset-version NAME`** (default `current-build`) — the
  export subdir name. Often something like `sentinel-v2-gold-2026-08`.

**The ready/skipped sources filter** (lines 689-699): a source
is "ready" if `data/preprocessed/<source>/` exists; otherwise
it's added to `skipped_sources` with reason `"preprocessed dir
not found"`. The chunked export only operates on ready sources.

**`chunk_export` orchestrates all 4 writers** — it's defined in
`export/chunker.py:151` (237 LOC, fully implemented). The export
returns an `ExportManifest` dataclass with `n_contracts`,
`n_contracts_with_reps`, `n_shards`, `artifact_hash`.

**The `export/` subpackage is fully implemented** (7 modules, 1482
LOC total): labels → metadata → graph shards → token shards →
shard index → artifact hash (Fix A: written before manifest.json).
`SentinelDatasetExport` at `export/export.py:21` is the consumer-facing
API. The only CLI STUB is `_run_label` (Stage 3); export is real.

---

## §11 `_run_freshness` (752-758) — The Thin Utility

```python
# Learning mode: Awareness | Thinnest handler in the file
def _run_freshness(args: argparse.Namespace) -> None:
    from sentinel_data.ingestion.freshness import run_freshness_check
    cfg = _load_config(args.config)
    data_dir = Path(args.config).parent / "data"
    print("[freshness] Checking source pins vs upstream HEAD + slither-analyzer version...")
    report = run_freshness_check(cfg, data_dir)
    print(report)
```

**The whole handler is 4 lines.** It:
1. Lazily imports `run_freshness_check`.
2. Loads config + resolves data_dir.
3. Prints a one-line header.
4. Calls the function and prints the report.

**`freshness.py`** (covered in Session 5) checks:
- Are the git pins in `config.yaml` still valid? (Has the
  upstream HEAD moved past the pin?)
- Is the installed `slither-analyzer` version compatible with
  the version expected by `verification/slither_runner.py`?
- (Anything else the freshness module decides to check.)

**Why this is a utility, not a stage:** freshness doesn't write
artifacts. It just prints a report. It doesn't fit the pipeline
DAG (per `dvc.yaml`). The CLI exposes it as a subcommand for
convenience, but it lives outside the 9-stage flow.

---

## §12 The 3-Layer Defense Pattern in `_run_verify` and `_run_analyze`

Both `_run_verify` and `_run_analyze` are **orchestrators** — they
sequence multiple sub-tools and combine their results. The pattern
is identical:

```
for tool in [tool1, tool2, tool3, ...]:
    if args.dry_run: continue
    if args.only and args.only != tool: continue
    print(f"\n  [{i}/N] {tool}")
    result = run_X(data_dir, ...)
    print(f"    {result.summary}")
```

**Why this pattern is good:**
- Each sub-tool is independently testable.
- Each can be skipped via a flag (verify) or `--only` (analyze).
- The orchestrator doesn't know the internals of each sub-tool —
  it just calls the public `run_X` entry point.
- The dry-run mode covers all sub-tools uniformly.

**Where it could be improved:** the orchestrators have a lot of
boilerplate (print headers, parse config, resolve corpus paths).
A v2.1 refactor could extract a `StageOrchestrator` base class.
(Not on the roadmap; the v2 build is focused on correctness, not
abstraction.)

---

## §13 3 Things to Lock In (per global P10-C)

1. **The `_run_*` handler template is uniform: lazy import → load
   config → resolve `data_dir` → print header → branch on args →
   dispatch.** What varies per stage is the **per-stage args** (e.g.
   `--workers` for preprocess, `--limit`/`--force`/`--emit-cfg` for
   represent, `--strict`/`--skip-*` for verify) and the **per-stage
   Python entry point** (e.g. `preprocess_all` vs. `represent_source`
   vs. `run_audit`).

2. **`_run_verify` is the only stage that returns an int exit
   code** (line 482-486). `--strict` turns a gate FAIL into exit
   code 1 (CI-friendly). All other stages return `None` on
   success — the orchestrator `_handle_run` only checks `isinstance(result, int)`.

3. **The cache-eviction fix for the Run 8 v8/v9 silent mix lives
   in `_run_represent`** at lines 202-204 + 225:
   `check_and_evict(...)` per source + `write_registry(...)` at
   the end. This is the operational reason the Run 9 v11 results
   are trustworthy. If you remove this code, you re-introduce
   the Run 8 failure mode.

---

## §14 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> `_run_split` builds `Contract` objects from `.labels.json` files
> with a specific tier-inference rule (line 274-277: take the
> first positive class's tier). What breaks if a contract is
> multi-label and its positive classes span different tiers
> (e.g. one Reentrancy=T0, one Reentrancy=T2)?

### Q2 [Pattern]

> The orchestrators (`_run_verify`, `_run_analyze`) and the
> individual stage handlers share the same dry-run pattern but
> express it differently. Some return early after printing a
> one-line message; `_run_represent` returns inside the per-source
> loop after a multi-line plan. Why the variation?

### Q3 [Portable🔵]

> `_run_export` resolves 4 args from CLI (`--split-version`,
> `--output-dir`, `--shard-size`, `--dataset-version`) with
> defaults coming from 3 different sources (CLI defaults,
> `config.yaml:export_shard_size`, the `dataset_version` default
> string). What's the general principle of CLI-arg-vs-config
> precedence, and what could go wrong if the precedence is
> inconsistent?

### Q4 [Mechanism]

> If you add a new `--per-class-cap` flag to `_run_split`, list
> every place in `cli.py` you must update (analogous to Q4 from
> Session 02, but for a per-stage arg rather than a new stage).

---

## §15 Connections (what to study next)

- **Session 04 — CLI orchestrator:** `cli.py:700-1037` — the
  `_STAGE_FN` dispatch table (line 745), the `_build_parser`
  function (line 761, ~220 LOC), and the `main()` entry point
  (line 1016). Plus the `freshness` utility subparser.
- **Session 05 — Ingestion overview:** `sentinel_data/ingestion/ingest.py`
  + `manifest.py` + `freshness.py`. Now that you know how
  `_run_ingest` dispatches, you can read the actual `ingest_all`
  + `ingest_source` functions.

---

## §16 What "got it" looks like for Session 03

- [ ] Sketch the 5-step handler template with one example per
      stage.
- [ ] Name the unique args for each stage (`preprocess`:
      `--workers`/`--sample`/`--retry-failed`; `represent`:
      `--limit`/`--force`/`--emit-cfg`; `verify`:
      `--strict`/`--skip-*`; etc.).
- [ ] Explain why `_run_label` is a STUB and what "implementing
      it" would entail (per module P104, just teach the STUB).
- [ ] Identify the cache-eviction fix at lines 202-204 + 225 as
      the operational reason Run 9 v11 is trustworthy.
- [ ] Predict what `_run_register` does if the manifest path
      doesn't exist (line 337-339).
- [ ] Trace the path of a `--corpus v1.4-bccc --baseline-version
      current-build` invocation through `_resolve_corpus_paths`.

---

**Next:** answer Q1–Q4 in chat. I'll fill in **Your answers** at
§0.6, gap-fill (P2) if needed, and update `session_log.md`. After
Q&A, we move to Session 04.
