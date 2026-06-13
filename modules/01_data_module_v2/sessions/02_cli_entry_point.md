# Session 02 — CLI Entry Point: `__init__.py` + `cli.py:1-200`

**Date:** 2026-06-13
**Session number:** 02 of 36
**Mode:** Understand + Master (the bootstrap is a non-obvious mechanism)
**Estimated study time:** 45 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0 Recap of Session 01 + Model Answers to Q1–Q4

> **Why this section is here:** the user asked for answers + new learning
> in the same doc. So this section provides model answers to Session 01's
> Q1–Q4 (what an experienced learner would say), then transitions into
> Session 02's new material.
>
> When you respond to Q1–Q4 in chat, I'll fill in the **Your answers**
> section with your actual answers + any gap-fill (P2). The model
> answers below are the expected reasoning pattern.

### §0.1 Recap of Session 01

Session 01 established three mental models: **(1)** the two `class_names()`
orders diverge at indices 5, 8, 9; **(2)** the 9-stage pipeline runs
ingest → preprocess → represent → label → verify → split → register →
analyze → export; **(3)** the 3-layer configuration (config.yaml +
dvc.yaml + pyproject.toml) is the single source of truth, with each
file playing a distinct role. Stage 7 (export) is a STUB. Stage 3
label CLI is a STUB.

### §0.2 Model answer to Q1 [Pattern]

> **Q:** Why does the data_module have two different `class_names()`
> orders, and what would break if you changed the representation order
> to match the labeling order?

**Model answer:**

Two reasons for the divergence:

1. **Backward compatibility with existing model checkpoints.** The Run 9 v11
   classifier head is `nn.Linear(256, 10)`. The 10 weight rows correspond
   to the representation order. Changing the order would silently
   re-map every class — the model would predict `Reentrancy` when the
   input is `CallToUnknown`, etc. Every existing checkpoint
   (Runs 1–9) becomes invalid.
2. **v2 design intent diverged from v1.** When the v2 build added
   `TransactionOrderDependence` (which wasn't in v1) and re-thought
   `NonVulnerable` as a negative label rather than a class slot, the
   labeling taxonomy needed restructuring. The representation kept
   the v1 shape to preserve checkpoint compatibility.

**What would break if you changed representation order to match labeling:**

- All existing model checkpoints (Run 1–9) become silently wrong.
  The 5 critical tests (`byte-identical`, `BCCC regression`,
  `sentinel_dataset round-trip`, etc.) would all fail.
- The 10-class softmax output position would be misaligned with
  `data/processed/multilabel_index.csv` (the v1 column-order contract
  that the trainer reads by position).
- The `SentinelDataset` col-loading pipeline would still work
  (string-keyed lookups), but the inference output would be
  systematically wrong on every contract.

**Common misconception:** "We can just change the labeling order to
match representation." This would lose the v2 design clarity around
`NonVulnerable` (negative label, not a class slot) and
`TransactionOrderDependence` (newly added). The reverse merge is
possible but requires an ADR + a checkpoint-migration plan.

### §0.3 Model answer to Q2 [Mechanism]

> **Q:** If you add a new vulnerability class (say, `BadRandomness`) to
> the labeling taxonomy at the end, what 3+ files must you update?
> Which update would silently corrupt an existing checkpoint if you
> forgot it?

**Model answer:**

3+ files to update:

1. **`sentinel_data/labeling/schema/taxonomy.yaml`** — add
   `BadRandomness` to the class list (alphabetic position).
2. **`sentinel_data/representation/graph_schema.py:73-84`** — add
   `BadRandomness` to the `CLASS_NAMES` list in the representation
   order. Bump `NUM_CLASSES` to 11.
3. **`data/processed/multilabel_index.csv`** — the v1 column-order
   contract that the existing trainer reads by position. Add a new
   column for `BadRandomness`. The **column order** here is the
   silent-killer: if the new column is in a different position than
   the order in the Python source, the labels at the old positions
   shift and the training proceeds with misaligned data.

Additional updates (recommended):

- Bump `FEATURE_SCHEMA_VERSION` from `"v9"` to `"v10"` in
  `graph_schema.py`. This invalidates the cache (so old
  representations are evicted on next run) and signals the
  schema change to all consumers.
- Update the per-class verification patterns
  (`verification/patterns/*.yaml`) if the new class needs
  semantic-check coverage.
- Update the CLI help text (`cli.py:_build_parser`).

**The update that silently corrupts an existing checkpoint:** the
`multilabel_index.csv` column order. The trainer reads columns
by position, not by name. If you add a column at the end of the
taxonomy but the index CSV is regenerated with the new column in
a different position, the labels at the old positions will be
misaligned. Training would proceed (no error thrown), but the
labels would be assigned to the wrong class. This is exactly the
bug that the v8/v9 schema test (`test_byte_identical_regression.py`)
guards against.

**Common misconception:** "Updating the Python source is enough."
The CSV file is the data-layer contract; it can drift from the
Python schema if not regenerated carefully. The `versioner.py`
module exists to detect this drift (evicts stale cache entries on
schema version mismatch).

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** The sentinel-data package has a "one-way dependency" rule: it
> never imports from sentinel-ml. What's the general software-
> engineering principle, and where else does this pattern appear?

**Model answer:**

The general principle: **stable-dependencies principle** (Robert
Martin) / **acyclic dependencies principle** / **dependency
inversion**. Lower-level modules should not depend on higher-level
modules. The more stable / more general module should be at the
bottom of the dependency graph.

**Where this pattern appears:**

| Domain | "Bottom" doesn't depend on "top" |
|--------|----------------------------------|
| OS kernels | Kernel doesn't import from user-space apps |
| Standard library | `os`, `sys`, `json` don't import from PyPI |
| Microservices | Each service owns its data; no cross-service DB writes |
| Clean / Hexagonal architecture | Inner hexagon (entities, business rules) is dependency-free; outer adapters depend inward |
| Database read replicas | The DB doesn't call the app |
| CI/CD | The build pipeline doesn't call the app at test time |
| Test fixtures vs. production code | Tests can import prod, prod can't import tests (similar shape) |
| **Source code vs. learning workspace** | **`ml/src/` doesn't depend on `learning_sentinel/`** (the read-only source doesn't know about our study material) |

**Why it matters for `sentinel-data`:** if `sentinel-data` could
import from `sentinel-ml`, then a change to `sentinel-ml` (a model
checkpoint update, a trainer refactor) could break the data
pipeline. The data pipeline would have to be re-tested every time
the model code changes. With the one-way rule, the data pipeline
is independently testable and versionable — Run 9 v11 used the
same v9 schema regardless of how the trainer evolved.

**Common misconception:** "One-way dependency = no sharing." You
can SHARE data and types, just not call across the boundary.
The schema constants (`FEATURE_SCHEMA_VERSION` etc.) are defined
in `ml/src/preprocessing/graph_schema.py` and re-exported via
thin-adapters; both sides agree on the contract without either
side importing from the other.

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** The `pipeline.min_viable_corpus` block has `total_contracts_min:
> 4000` AND `per_class_positive_min_major: 300` AND
> `per_class_positive_min_minor: 100`. The current corpus has ~22,356
> contracts. Why are all three floors needed?

**Model answer:**

Being above 4000 total does **NOT** guarantee per-class minimums. A
22,356-contract corpus could be 21,000 NonVulnerable + 1,000 spread
across the 9 vuln classes, and many classes would have < 100 positives.

- **`total_contracts_min: 4000`** is a **sanity check** — catches
  "we don't have enough data at all."
- **`per_class_positive_min_major: 300`** is a **class-imbalance
  guard for the 3 most-studied classes** (Reentrancy, DoS, IntegerUO).
  The 300 threshold reflects the empirical learning curve: below ~300
  positives, the model can't learn discriminative features.
- **`per_class_positive_min_minor: 100`** is a **less-strict
  guard for the 7 rarer classes**. Some classes (e.g.
  TransactionOrderDependence, BadRandomness) are inherently rarer
  in the wild; 100 is enough to learn something, even if the F1
  ceiling is lower.

**The current corpus:** the v2-readiness report shows that with
the SolidiFI+DIVE subset (~22,356 contracts), the per-class gate
verdicts are: **2 VERIFIED** (MishandledException, TransactionOrderDependence),
**5 PROVISIONAL** (Reentrancy, Timestamp, GasException, IntegerUO, UnusedReturn),
**3 BEST-EFFORT** (CallToUnknown, ExternalBug, DenialOfService). The
3 BEST-EFFORT classes are below their per-class floors — `CallToUnknown`
in particular has only 39 contracts (SolidiFI only), well below both
the 100-minor and the 300-major floor.

**That's why the corpus fails Gate 5 (no FAIL classes, but not all
VERIFIED or PROVISIONAL).** The friend-review `merge_rules` is the
response: if `CallToUnknown_verified_count < 300`, `pause_and_ask_human`
to merge it into `ExternalBug` (reversible). This is a structural
response to the fact that the corpus cannot satisfy all 10
per-class floors simultaneously with the current source mix.

**Common misconception:** "22,356 > 4000, so we're fine." The
total floor is just one of three; per-class starvation is the real
risk. The 22,356 total includes a lot of NonVulnerable negatives
(capped at 3:1 by `positive_ratio_max`).

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** _pending_

**Your answer to Q2:** _pending_

**Your answer to Q3:** _pending_

**Your answer to Q4:** _pending_

---

## §1 The Pair at a Glance (per global P5)

| File | LOC | Role |
|------|-----|------|
| `sentinel_data/__init__.py` | 57 | Package docstring (the architecture in prose) + `__version__` |
| `sentinel_data/cli.py:1-200` | 200 | Module docstring + `sys.path` bootstrap + dispatch data + helpers + the first stage handler template |

This is the **entry-point layer**. The package docstring explains
the *what* and *why*; the CLI module provides the *how*.

---

## §2 `__init__.py` — The Architecture in Prose

The package docstring (lines 1–56) is unusually rich for an
`__init__.py`. It explicitly documents the **5-stage + 3 cross-cutting**
structure, the **one-way dependency rule**, the **reproducibility
contract**, and the **thin-adapter contract**.

```python
# Learning mode: Understand | Docstring is the architecture in prose
"""
sentinel-data — SENTINEL data engineering module.

The package is a 5-stage pipeline + 3 cross-cutting systems:
    raw → ingest → preprocessed → preprocess → labeled → label →
    verified → represent → representations → split → splits
    + 3 cross-cutting: verification, registry, analysis
    + 1 final: export (Stage 7)
"""
```

**Key facts extracted from the docstring** (per `__init__.py:31-46`):

- **One-way dependency rule:** `sentinel-data` has a one-way
  dependency on nothing in `sentinel-ml`. Training consumes
  artifacts produced here; training code never feeds back.
  *(This is what makes the data pipeline independently testable,
  versionable, and reusable.)*
- **Reproducibility contract:** every artifact is content-addressed
  by SHA-256 and stamped with `schema_version` +
  `extractor_version` in a JSON sidecar. Two runs with the same
  config produce byte-identical outputs.
- **Thin-adapter contract:** the `representation` subpackage
  re-exports the PyG graph and CodeBERT token logic from
  `ml/src/preprocessing/` and `ml/src/data_extraction/` via
  ~10-line re-export files. Bug fixes in graph/token extraction
  apply once (in `ml/`) and propagate automatically to the new
  path. Stage 7 deletes the wrappers and rebinds the import.

The version is `__version__ = "0.1.0"` (line 57). The version
of artifacts it produces is separate (`schema_version="v9"`,
`extractor_version="v2.1-windowed-gcb"`); changing the package
implementation does NOT automatically invalidate cached
representations unless the version stamps change.

---

## §3 `cli.py:1-50` — The CLI Module Docstring

The CLI module docstring (lines 1–50) explains the design
choices for using a CLI instead of a Python library:

```python
# Learning mode: Understand | Why a CLI, not just a Python library
"""
Why a CLI (and not just a Python library):

* Operations: users running the pipeline don't want to write Python;
  they want `sentinel-data run --stage represent`.
* Configuration loading: the CLI centralises config loading, path
  resolution, dry-run mode, and argument validation in one place.
* Future DVC integration: DVC will call the CLI, not the Python API,
  for clean separation of concerns.

Stages (in pipeline order):
    ingest, preprocess, represent, label, verify,
    split, register, analyze, export
"""
```

**Three design choices extracted from the docstring:**

1. **Operations:** users want commands, not Python imports.
2. **Config centralization:** the CLI is the single place that
   loads `config.yaml`, resolves `data_dir`, sets `dry-run` mode,
   and validates arguments.
3. **DVC integration:** DVC will invoke the CLI (not the Python
   API) for clean separation of concerns. Per `dvc.yaml:3-4`:
   `cmd: sentinel-data ingest --config config.yaml`.

---

## §4 `cli.py:62-68` — The `sys.path` Bootstrap (Master🔵)

```python
# Learning mode: MASTER🔵 | The ONLY place in production code that mutates sys.path
_HERE = Path(__file__).resolve()
_DATA_DIR = _HERE.parent.parent          # sentinel/Data/
_REPO_ROOT = _DATA_DIR.parent            # sentinel/
_ML_ROOT   = _REPO_ROOT / "ml"
for _p in (_REPO_ROOT, _ML_ROOT):
    if str(_p) not in sys.path:
        sys.path.insert(0, str(_p))
```

**Why this exists** (per docstring lines 36-49):

> "the representation subpackage uses a thin-adapter pattern:
> `sentinel_data.representation` re-exports the graph and tokenizer
> code from `ml/src/preprocessing/` and `ml/src/data_extraction/`.
> Without these paths on `sys.path`, the re-exports fail with
> `ModuleNotFoundError` whenever the CLI is invoked from outside
> the repo root (e.g. via an installed entry-point, a Docker
> container with a different CWD, or a CI runner)."

**The contract** (lines 47-49):

> "This is the only place in the production code where `sys.path`
> is manipulated; tests use `conftest.py` instead. The two
> strategies are intentionally parallel: the CLI must work
> without pytest, the test suite must work without the CLI."

**What breaks if you remove this bootstrap** (think through this
with module P107 in mind):

- Thin-adapter imports from `ml/src/preprocessing/graph_schema.py`
  fail with `ModuleNotFoundError` when the CLI is invoked from
  outside `~/projects/sentinel/` (e.g. `cd /tmp && sentinel-data
  ingest`).
- Docker container with a different CWD breaks the CLI.
- CI runner with a clean checkout breaks.
- The `sentinel-data` package can NEVER be installed as a
  standalone PyPI package (always requires `ml/` on `PYTHONPATH`).

The thin-adapter pattern (covered in Session 12) **depends on this
bootstrap**. It is the only reason the new path can re-export from
`ml/` without forcing a re-install or a sysadmin-level environment
fix.

**Why it's done at line 62 (before any imports of `sentinel_data.*`):**
Python evaluates `sys.path` lazily — the first time anything tries
to import from `ml.src.preprocessing.*`, the import system consults
`sys.path`. If we did the bootstrap *after* the imports at line 52-55,
the imports would already have failed.

**Why the underscore prefix:** `_HERE`, `_DATA_DIR`, `_REPO_ROOT`,
`_ML_ROOT` — these are module-level "private" names (the leading
underscore). They are not part of the public API.

**Why `insert(0, ...)` not `append(...)`:** insert at position 0
means "search this first." This matters when the user has a
different `ml/` package on their Python path (e.g. installed
globally via pip) — the local `~/projects/sentinel/ml/` takes
precedence.

**Why the loop instead of two separate `insert` calls:** DRY
(Don't Repeat Yourself). The loop body is the same for both paths.

---

## §5 `cli.py:71-93` — The Dispatch Data (Awareness)

```python
# Learning mode: Awareness | Single source of truth for stage list + per-stage help
STAGES: list[str] = [
    "ingest", "preprocess", "represent", "label", "verify",
    "split", "register", "analyze", "export",
]

STAGE_DESCRIPTIONS: dict[str, str] = {
    "ingest":     "Pull raw .sol contracts from all enabled sources with SHA-256 manifests",
    "preprocess": "Flatten + two-pass compile + dedup@0.85 + normalize + segment + version-bucket",
    "represent":  "Extract v9 graph (.pt) + windowed token files (GraphCodeBERT, [4,512])",
    "label":      "Apply crosswalk YAMLs; merge labels; flag 99%% co-occurrence pairs",
    "verify":     "AST semantic checks + tool corroboration + BCCC Phase 5 regression test",
    "split":      "Deterministic train/val/test splits (4 strategies, 2-pass with dedup_enforcer, NonVulnerable 3:1 cap)",
    "register":   "Register a dataset version in the SQLite catalog (4 base + 2 system tables, YAML mirror)",
    "analyze":    "Feature distribution + complexity proxy risk + co-occurrence matrix",
    "export":     "Shard export to sentinel-ml; predictor tier-threshold fix; EMITS edge fix",
}
```

**Why this data lives at module scope (not inside `_build_parser`):**

- **Other modules import it.** `_handle_run` (line 980) uses
  `STAGES.index(args.from_stage)` to compute the starting offset.
  `STAGE_DESCRIPTIONS` is used in `_handle_run` (line 987), the
  per-stage handlers (e.g. line 116), and `sentinel_data/README.md`
  references the same strings.
- **It's the single source of truth for "which stages exist"** —
  if you add a 10th stage, you add it here and the dispatch table
  (`_STAGE_FN` at line 745) and the DVC DAG and the README all
  need to follow.

**The `%%` in `"label"` description:** `99%%` is a format-string
escape for an actual `%` character. The `%%` is needed because
`STAGE_DESCRIPTIONS` is later used inside a `%`-format call
(likely a `print(f"...")` context). If the `%%` were `%`, Python
would try to interpret the next character as a format specifier.

---

## §6 `cli.py:96-106` — The Helpers (Understand)

```python
# Learning mode: Understand | Lazy imports keep the CLI startup fast
def _load_config(config_path: str) -> dict:
    import yaml  # type: ignore[import]
    with open(config_path) as f:
        return yaml.safe_load(f)

def _default_config() -> str:
    """Find config.yaml relative to the CLI entry point."""
    here = Path(__file__).resolve().parent.parent  # Data/
    candidate = here / "config.yaml"
    return str(candidate) if candidate.exists() else "config.yaml"
```

**`_load_config` is lazy-imported.** The `import yaml` is inside
the function body, not at the top of the file. Why? Because
`pyyaml` is the **only** required core dependency (per
`pyproject.toml:12`). If the user has installed just the core
group and never calls a stage that needs YAML (e.g. they invoke
`sentinel-data freshness` which doesn't load `config.yaml`),
`yaml` is never imported. This keeps the CLI startup fast and
the import graph minimal.

**`_default_config` resolution order:**

1. `Path(__file__).resolve().parent.parent` = `sentinel/data_module/`
2. Check if `sentinel/data_module/config.yaml` exists.
3. If yes, return it. If no, fall back to the string `"config.yaml"`.
4. Python's `open()` will then look in the CWD.

The fallback to CWD is for **Docker and CI scenarios** where the
config file is mounted at a different path. The CLI doesn't crash
if the config is missing — it just gives argparse a chance to
parse and validate other args first, and a clear error message
appears when the file is actually opened.

**The type annotation:** `def _load_config(config_path: str) -> dict`
is intentionally simple. `dict` is a loose return type; the
actual config is a deeply nested dict (per `config.yaml:32-100+`).
Adding precise types (e.g. `TypedDict`) would be a v2.1 enhancement
(captured in module P104 STUB honesty — don't speculate on the
unwritten types).

---

## §7 `cli.py:111-128` — The Stage Handler Template (`_run_ingest`)

```python
# Learning mode: Understand | The template; Session 03 covers all 9 handlers in detail
def _run_ingest(args: argparse.Namespace) -> None:
    from sentinel_data.ingestion.ingest import ingest_all, ingest_source  # ← lazy import
    cfg = _load_config(args.config)
    data_dir = Path(args.config).parent / "data"

    print(f"[ingest] {STAGE_DESCRIPTIONS['ingest']}")
    print(f"  config : {args.config}")

    source = getattr(args, "source", None)
    if source:
        print(f"  source : {source}")
        if args.dry_run:
            print("  (dry-run — no files written)")
        ingest_source(source, cfg, data_dir, dry_run=args.dry_run)
    else:
        if args.dry_run:
            print("  (dry-run — no files written)")
        ingest_all(cfg, data_dir, dry_run=args.dry_run)
```

**The 5-step template** (this is the shape every `_run_*` handler
follows):

1. **Lazy import** the per-stage functions from the subpackage
   (`from sentinel_data.ingestion.ingest import ingest_all, ingest_source`).
   Same DRY-of-startup principle as `_load_config`.
2. **Load config** via `_load_config(args.config)`.
3. **Resolve data_dir** as `Path(args.config).parent / "data"`.
   The convention: `data/` is always a sibling of `config.yaml`.
4. **Print the stage header** using `STAGE_DESCRIPTIONS[stage_name]`.
5. **Branch on optional args** (e.g. `--source`, `--dry-run`) and
   dispatch to the right per-stage function.

**`getattr(args, "source", None)` vs `args.source`:** the former
returns `None` if the attribute doesn't exist; the latter raises
`AttributeError`. `getattr` is used because per-stage parsers
add `--source` only when it's a valid option (some stages don't
have per-source subcommands). It's a defensive pattern that lets
the same handler code work across stages with different arg sets.

**The `dry_run` check:** the handler prints a "(dry-run)" message
AND passes `dry_run=True` to the per-stage function. Two layers of
defense: the user sees the message; the function refuses to write.

**Why `ingest_all` vs `ingest_source`:** single-source mode is for
debugging and re-running a specific source. All-sources mode is
for full pipeline runs. The CLI defaults to all-sources when
`--source` is omitted.

---

## §8 Where the Real Action Lives (Pointer to Session 03)

The dispatch table that maps `STAGES[i]` → the actual handler
function is at `cli.py:745-756` (Session 03 territory):

```python
# Learning mode: Awareness | Just a dict; covered in Session 03
_STAGE_FN = {
    "ingest":     _run_ingest,
    "preprocess": _run_preprocess,
    "represent":  _run_represent,
    "label":      _run_label,
    "verify":     _run_verify,
    "split":      _run_run_split,
    "register":   _run_register,
    "analyze":    _run_analyze,
    "export":     _run_export,
    "freshness":  _run_freshness,
}
```

The arg parser (`_build_parser` at line 761) is the largest single
function in `cli.py` (~220 LOC). Session 03 walks through both the
dispatch table and the parser.

---

## §9 3 Things to Lock In (per global P10-C)

1. **The `sys.path` bootstrap at `cli.py:62-68` is the only
   place in production code that mutates `sys.path`.** It exists
   to make the thin-adapter pattern work from any CWD (Docker, CI,
   installed entry-point). Removing it breaks `sentinel-data` for
   any environment outside the repo root.

2. **`STAGES` (line 71) is the single source of truth for which
   stages exist.** `_STAGE_FN` (line 745), `dvc.yaml`, and
   `README.md` must all follow. If you add a 10th stage, you must
   add it here first.

3. **The `_run_*` handler template is: lazy import → load config →
   resolve `data_dir` → print header → branch on args → dispatch
   to per-stage function.** Every stage follows this 5-step shape.
   Session 03 will read all 9 handlers in this template's shoes.

---

## §10 Abbreviations introduced this session (per global P12)

- **CWD** — Current Working Directory (the directory a process was launched from)
- **DRY** — Don't Repeat Yourself (a software design principle)
- **entry-point** — a CLI command installed by a Python package (e.g. `sentinel-data` is the entry point declared in `pyproject.toml:51`)
- **`%%` in format strings** — the literal escape for a `%` character in old-style `%` string formatting (vs. f-strings)

---

## §11 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> What breaks if the `sys.path` bootstrap at `cli.py:62-68` is
> removed? List 3 concrete failure modes.

### Q2 [Pattern]

> The `_run_ingest` handler uses `getattr(args, "source", None)`
> instead of `args.source`. Why is the defensive `getattr` form
> used here, and what would happen if the per-stage parser forgot
> to add `--source` for a stage that doesn't support per-source
> runs?

### Q3 [Portable🔵]

> The CLI uses **lazy imports** for `yaml` (in `_load_config`) and
> for per-stage functions (e.g. `from sentinel_data.ingestion.ingest
> import ingest_all`). What's the general principle of lazy imports,
> and what are the trade-offs vs. eager top-of-file imports?

### Q4 [Mechanism]

> The `_STAGE_FN` dispatch table at line 745 maps stage names to
> handler functions, but it lives far from the `STAGES` list at
> line 71. If you add a 10th stage (e.g. `drift_report`), list
> every place in `cli.py` that you must update for the new stage
> to be callable from `sentinel-data <stage>`.

---

## §12 Connections (what to study next)

- **Session 03 — CLI handlers + dispatch:** `cli.py:200-700`
  (per-stage handlers) + `cli.py:700-1037` (dispatch table at
  line 745, `_build_parser` at line 761, `main()` at line 1016).
  You'll see all 9 stage handlers in the template shape from
  Session 02's §7, the dispatch dict, and the argparser that
  wires `--source` / `--workers` / `--sample` / `--limit` /
  `--emit-cfg` / etc. per stage.
- **Session 04 — CLI orchestrator:** the `_handle_run` function
  (line 980) that walks multiple stages in sequence, plus the
  `freshness` utility.

---

## §13 What "got it" looks like for Session 02

- [ ] Explain why the `sys.path` bootstrap is the **only** place
      `sys.path` is mutated in production code.
- [ ] Predict 3 concrete failure modes if the bootstrap is removed.
- [ ] Sketch the 5-step template that every `_run_*` handler
      follows, with one example from `_run_ingest`.
- [ ] Explain why `STAGES` is at module scope, not inside
      `_build_parser`.
- [ ] List every place in `cli.py` you'd touch to add a 10th
      stage (Q4 answer).

---

**Next:** answer Q1–Q4 in chat. I'll fill in the **Your answers**
section at §0.6, gap-fill (P2) if needed, and update
`session_log.md`. After Q&A, we move to Session 03.
