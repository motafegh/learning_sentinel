# Session 04 — CLI Orchestrator (`cli.py:700-1053`)

**Date:** 2026-06-14
**Session number:** 04 of 36
**Mode:** Understand (dispatch table + parser + entry point — structural, not mechanism-heavy)
**Estimated study time:** 30 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0 Recap of Session 03 + Model Answers to Q1–Q4

### §0.1 Recap of Session 03

Session 03 read `cli.py:200-700` — all 9 per-stage handlers. Key concepts:
**(1)** every handler follows the same 5-step template (lazy import → load config
→ resolve data_dir → print header → branch on args); **(2)** `_run_verify` is
the only handler returning an int exit code (`--strict`); **(3)** cache eviction
in `_run_represent` (lines 202-204 + 225) is the Run 8 v8/v9 silent-mix fix;
**(4)** `_run_split` builds `Contract` objects from `.labels.json` with a
tier-inference heuristic (first positive class's tier); **(5)** `_run_label`
is a CLI STUB (7 lines).

### §0.2 Model answer to Q1 [Mechanism]

> **Q:** `_run_split` builds `Contract` objects from `.labels.json` files
> with a specific tier-inference rule (line 274-277: take the first positive
> class's tier). What breaks if a contract is multi-label and its positive
> classes span different tiers (e.g. one Reentrancy=T0, one Reentrancy=T2)?

**Model answer:**

The `Contract.tier` field is a **single string**, not a list. The splitter
stratifies by tier — the contract ends up in exactly one tier bucket. The
inference rule at line 274-277 is:

```python
for cls, entry in lj.get("classes", {}).items():
    if entry.get("value") == 1:
        tier = entry.get("tier") or "T0"
        break
```

This iterates over `classes` in **dict insertion order** (Python 3.7+),
which is the order the merger wrote the `.labels.json`. If a contract
has Reentrancy=T0 first and DoS=T2 second, the tier is `T0` and the
DoS tier is silently discarded.

**What breaks:**

1. **Stratified split misplacement.** A contract that should be in the
   T2 bucket (because one of its labels is T2) ends up in T0. The
   stratified splitter uses tier as a stratification column. This
   means the T0 split gets slightly more contracts and the T2 split
   gets slightly fewer. The effect is small (a few contracts) but
   the stratification is no longer faithful to the source data.

2. **Verification correlation is wrong.** If the contract later
   appears in a verification report, its tier determines which
   stratification bucket the FP estimator uses. A T0 contract
   appearing in T2 changes the FP rate estimate for both tiers.

3. **NonVulnerable cap interaction.** The cap is stratified by
   source. Tier misplacement doesn't directly affect the cap, but
   it affects the per-tier stats that the cap's implementation
   relies on for balanced subsampling.

**Common misconception:** "The merger guarantees one tier per contract."
No — the merger assigns tiers per-class. A multi-label contract
gets different tiers for different classes. The splitter's
single-tier inference is a simplification that trades accuracy
for implementation simplicity.

### §0.3 Model answer to Q2 [Pattern]

> **Q:** The orchestrators (`_run_verify`, `_run_analyze`) and the individual
> stage handlers share the same dry-run pattern but express it differently.
> Some return early after printing a one-line message; `_run_represent`
> returns inside the per-source loop after a multi-line plan. Why the
> variation?

**Model answer:**

The variation reflects **the cost of the dry-run plan itself:**

- **Cheap handlers** (`_run_ingest`, `_run_label`, `_run_freshness`):
  the dry-run prints one line and returns. The plan is trivial —
  there's nothing to enumerate.

- **Moderate handlers** (`_run_verify`, `_run_analyze`): the dry-run
  needs to resolve corpus paths and print which tools will run. This
  is a few lines but doesn't require reading any data.

- **Expensive handlers** (`_run_represent`, `_run_preprocess`): the
  dry-run needs to enumerate which files will be processed, check
  cache hits, and print a per-source breakdown. This requires
  reading the manifest/registry. The dry-run returns inside the
  loop because it's iterating over sources anyway.

The principle: **dry-run cost should be proportional to what the
user learns from it.** A one-line "would run ingest" is useless
for represent (where the user wants to know "how many files will
be processed?"). But printing a full file list for ingest is
wasteful when the user just wants to confirm the source is correct.

**Common misconception:** "All dry-runs should be uniform." Uniformity
is simpler but less useful. The current variation is a pragmatic
trade-off.

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** `_run_export` resolves 4 args from CLI with defaults coming from
> 3 different sources (CLI defaults, `config.yaml:export_shard_size`,
> the `dataset_version` default string). What's the general principle
> of CLI-arg-vs-config precedence, and what could go wrong if the
> precedence is inconsistent?

**Model answer:**

The general principle: **CLI flags override config, config overrides
defaults.** This is the standard Unix convention (think `git` with
`~/.gitconfig`).

In `_run_export`:
- `--shard-size` defaults to `None` in argparse → falls back to
  `config.yaml:export_shard_size` → falls back to `5000`
- `--dataset-version` defaults to `"current-build"` → used as-is
  (no config override)
- `--split-version` defaults to `1` → used as-is
- `--output-dir` defaults to `None` → constructed from `dataset_version`

The precedence chain is: **argparse → config.yaml → hardcoded default**.

**What goes wrong if inconsistent:**
- If some args use config-first and others use CLI-first, the user
  can't predict behavior. "I set `shard_size: 10000` in config but
  the CLI used 5000" → confusion.
- If the config key name doesn't match the CLI flag name (e.g.
  `export_shard_size` vs `--shard-size`), the mapping is fragile
  and undocumented.
- If the fallback chain is broken (argparse default is non-None but
  the config key is also set), the argparse default wins silently —
  the config value is ignored without warning.

**Where this pattern appears everywhere:**
- `git` (CLI flags → `~/.gitconfig` → built-in defaults)
- `docker` (CLI flags → `~/.docker/config.json` → defaults)
- `nginx` (CLI flags → `nginx.conf` → compiled defaults)
- `pytest` (CLI flags → `pyproject.toml` → built-in defaults)

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** If you add a new `--per-class-cap` flag to `_run_split`, list
> every place in `cli.py` you must update.

**Model answer (4 places in `cli.py`):**

1. **`_build_parser`** — inside the `if stage == "split":` block
   (around line 911-923), add:
   ```python
   sp.add_argument(
       "--per-class-cap", type=float, default=3.0, metavar="RATIO",
       help="Per-class NonVulnerable cap (overrides config default)",
   )
   ```

2. **`_handle_run`** — in the `stage_args = argparse.Namespace(...)`
   block (lines 1011-1018), add the default:
   ```python
   per_class_cap=3.0,
   ```
   This ensures `sentinel-data run` passes a valid default for the
   new arg even though `run` doesn't have a per-stage parser.

3. **`_run_split` handler** — change line 384 from:
   ```python
   apply_nonvulnerable_cap(splits, cap=args.nonvuln_cap, seed=args.seed)
   ```
   to:
   ```python
   cap = getattr(args, "per_class_cap", None) or args.nonvuln_cap
   apply_nonvulnerable_cap(splits, cap=cap, seed=args.seed)
   ```
   Using `getattr` because `run` mode injects a default Namespace
   that might not have the new attribute until the next deploy.

4. **Module docstring** — optional but recommended: update the
   stages list or add a note about the new flag.

**Common misconception:** "Just add it to `_build_parser` and the
handler." That misses `_handle_run`, which constructs a synthetic
Namespace for the `run` subcommand. Without updating it, `sentinel-data
run` would crash with `AttributeError` on `args.per_class_cap`.

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** _pending_

**Your answer to Q2:** _pending_

**Your answer to Q3:** _pending_

**Your answer to Q4:** _pending_

---

## §1 What This Chunk Covers (per global P5)

`cli.py:700-1053` — the **tail of the CLI**: the `_STAGE_FN` dispatch
dict, the `_build_parser` function (~220 LOC), the `_handle_run`
pipeline runner, and the `main()` entry point. This is the
**structural skeleton** that wires everything together.

| Component | Lines | Role |
|-----------|-------|------|
| `_STAGE_FN` | 761-772 | Dispatch dict: stage name → handler function |
| `_build_parser` | 777-991 | Argparse setup: 1 `run` subparser + 9 stage subparsers + 1 utility |
| `_handle_run` | 996-1027 | Pipeline runner: walk `STAGES[start_idx:]`, call each handler |
| `main()` | 1032-1053 | Entry point: parse args, dispatch to handler or `_handle_run` |

---

## §2 `_STAGE_FN` (761-772) — The Dispatch Dict (Awareness)

```python
# Learning mode: Awareness | Simple dict mapping stage names to handler functions
_STAGE_FN = {
    "ingest":     _run_ingest,
    "preprocess": _run_preprocess,
    "represent":  _run_represent,
    "label":      _run_label,
    "verify":     _run_verify,
    "split":      _run_split,
    "register":   _run_register,
    "analyze":    _run_analyze,
    "export":     _run_export,
    "freshness":  _run_freshness,
}
```

**10 entries** (9 pipeline stages + `freshness` utility). This is the
single mapping from "user types `sentinel-data <X>`" to "Python
function runs."

**Why it's at module scope:** `main()` at line 1043 does
`fn = _STAGE_FN.get(args.command)`. `_handle_run` at line 1008 does
`fn = _STAGE_FN[stage]`. Both need the dict available at call time.
Module scope is the simplest way.

**`freshness` is here but not in `STAGES`:** `freshness` is a utility,
not a pipeline stage. It's not in the `STAGES` list (line 71-81) and
`_handle_run` won't call it (it only iterates `STAGES`). But it's in
`_STAGE_FN` so `main()` can dispatch to it. The two data structures
serve different purposes:
- `STAGES` = "what runs in the pipeline DAG"
- `_STAGE_FN` = "what commands the CLI accepts"

---

## §3 `_build_parser` (777-991) — The Arg Parser (~220 LOC)

This is the **largest single function** in `cli.py`. It builds the
entire argparse tree.

### §3.1 Top-level structure

```python
# Learning mode: Understand | The parser has 3 layers: root → subparsers → per-stage args
def _build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="sentinel-data",
        description="SENTINEL data pipeline — build a verified multi-source ...",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    subparsers = parser.add_subparsers(dest="command", metavar="COMMAND")
    # ... 11 subparsers total
    return parser
```

**`dest="command"`** — argparse stores the chosen subcommand name
in `args.command`. This is how `main()` knows which handler to call.

**`formatter_class=RawDescriptionHelpFormatter`** — preserves
whitespace in the description. The multi-line pipeline description
at line 780-785 would be mangled by the default formatter.

### §3.2 The `run` subparser (791-801)

```python
# Learning mode: Understand | The only subparser that doesn't map to a stage
run_p = subparsers.add_parser("run", help="Run the full pipeline (or resume from a stage)")
run_p.add_argument("--from-stage", choices=STAGES, default=None)
run_p.add_argument("--config", default=_default_config())
run_p.add_argument("--dry-run", action="store_true")
```

`run` is special: it's not a stage, it's the **orchestrator**. It
iterates `STAGES[start_idx:]` and calls each handler. The
`--from-stage` flag lets the user skip earlier stages (useful when
you've already run ingest+preprocess and want to resume from
represent).

**`choices=STAGES`** — argparse validates the value against the
`STAGES` list. Invalid stage names get rejected at parse time, not
at runtime.

### §3.3 Per-stage subparser loop (803-982)

```python
# Learning mode: Understand | Conditional args per stage; same pattern as getattr in handlers
for stage in STAGES:
    sp = subparsers.add_parser(stage, help=STAGE_DESCRIPTIONS[stage])
    sp.add_argument("--config", default=_default_config())
    sp.add_argument("--dry-run", action="store_true")
    if stage in ("ingest", "preprocess", "represent"):
        sp.add_argument("--source", ...)
    if stage == "represent":
        sp.add_argument("--workers", ...)
        sp.add_argument("--limit", ...)
        sp.add_argument("--force", ...)
        sp.add_argument("--emit-cfg", ...)
    # ... similar blocks for preprocess, verify, split, register, analyze, export
```

**The pattern:** a loop over `STAGES` creates 9 subparsers. Each
gets `--config` and `--dry-run` (the universal args). Then
conditional blocks add stage-specific args.

**Why this is clean:** the parser shape mirrors the handler shape.
If a handler uses `getattr(args, "workers", 1)`, the parser adds
`--workers` in the `if stage == "preprocess":` block. If a handler
doesn't use an arg, it's not added to the parser.

**The `if stage in (...)` idiom** (line 808): three stages share
`--source`. Rather than three separate blocks, a single membership
test adds it once.

### §3.4 Per-stage arg summary

| Stage | Unique args |
|-------|------------|
| ingest, preprocess, represent | `--source NAME` |
| preprocess | `--workers N`, `--sample N`, `--retry-failed` |
| represent | `--workers N`, `--limit N`, `--force`, `--emit-cfg` |
| verify | `--strict`, `--semantic-limit-per-class N`, `--tool-limit-per-class N`, `--negative-limit N`, `--force-slither`, `--skip-tool-validator`, `--skip-fp-estimator`, `--skip-negative-checker` |
| split | `--version N`, `--seed N`, `--nonvuln-cap RATIO` |
| register | `--name NAME` (required), `--version N`, `--sources SRC...`, `--verification-report PATH`, `--retire-previous NAME` |
| analyze | `--only TOOL`, `--run-id ID`, `--corpus VERSION`, `--baseline-version VERSION` |
| export | `--dataset-version NAME`, `--split-version N`, `--output-dir PATH`, `--shard-size N` |

**Verify has the most unique args (8).** This reflects its role as
the most complex stage — 5 sub-checkers, each skippable, with
sample limits for smoke testing.

**Register is the only stage with a required arg** (`--name`).
All others have sensible defaults.

### §3.5 The `freshness` subparser (984-989)

```python
# Learning mode: Awareness | Utility, not a pipeline stage
fresh_p = subparsers.add_parser("freshness", help="Check source pin staleness + slither-analyzer version")
fresh_p.add_argument("--config", default=_default_config())
```

Minimal: just `--config`. The handler is 4 lines. This is the
simplest subparser in the file.

---

## §4 `_handle_run` (996-1027) — The Pipeline Runner

```python
# Learning mode: Understand | Walks STAGES[start_idx:], calls each handler with a default Namespace
def _handle_run(args: argparse.Namespace) -> None:
    start_idx = STAGES.index(args.from_stage) if args.from_stage else 0
    stages_to_run = STAGES[start_idx:]

    if args.dry_run:
        print(f"[run --dry-run] Would execute {len(stages_to_run)} stage(s):")
        for s in stages_to_run:
            print(f"  {s:<12}  {STAGE_DESCRIPTIONS[s]}")
        return

    failed = False
    for stage in stages_to_run:
        fn = _STAGE_FN[stage]
        stage_args = argparse.Namespace(
            config=args.config, dry_run=False, source=None,
            workers=1, sample=None, retry_failed=False,
            strict=False, semantic_limit_per_class=None, ...
        )
        result = fn(stage_args)
        if isinstance(result, int) and result != 0:
            failed = True
            print(f"  Stage '{stage}' returned exit code {result}; aborting run.")
            break
    if failed:
        sys.exit(1)
```

### §4.1 The `--from-stage` logic (line 997-998)

```python
start_idx = STAGES.index(args.from_stage) if args.from_stage else 0
stages_to_run = STAGES[start_idx:]
```

`STAGES` is `["ingest", "preprocess", "represent", "label", "verify",
"split", "register", "analyze", "export"]`. If `--from-stage
represent`, then `start_idx=2` and `stages_to_run=["represent", ...,
"export"]`. Ingest and preprocess are skipped.

**`STAGES.index()` is O(n) but n=9.** Not a performance concern.

### §4.2 The synthetic `stage_args` (lines 1011-1018)

```python
stage_args = argparse.Namespace(
    config=args.config, dry_run=False, source=None,
    workers=1, sample=None, retry_failed=False,
    strict=False, semantic_limit_per_class=None, ...
)
```

This is the **key design decision** of `_handle_run`: it constructs
a **default Namespace** for each stage, ignoring any per-stage CLI
args. When running via `sentinel-data run`, you can't pass
`--workers 4` to just the preprocess stage — all stages get
defaults.

**Why:** the `run` subparser only has `--from-stage`, `--config`,
and `--dry-run`. It doesn't have per-stage args. Adding them would
make `run`'s parser enormous and ambiguous (what if `--workers`
applies to preprocess but not represent?). The current design
accepts this limitation: `run` mode uses defaults, per-stage
mode lets you customize.

**The `isinstance(result, int)` check** (line 1022): only `_run_verify`
returns an int. All others return `None`. This is the only place
that checks for failure across stages. If any future stage returns
a non-zero int, `_handle_run` will abort the pipeline.

### §4.3 The dry-run mode (lines 1000-1004)

```python
if args.dry_run:
    print(f"[run --dry-run] Would execute {len(stages_to_run)} stage(s):")
    for s in stages_to_run:
        print(f"  {s:<12}  {STAGE_DESCRIPTIONS[s]}")
    return
```

Dry-run in `run` mode prints the **list of stages that would run**
and returns. It does NOT call the individual handlers' dry-run logic.
This is a deliberate choice: `run --dry-run` shows you the plan;
per-stage `--dry-run` shows you the per-file plan.

---

## §5 `main()` (1032-1053) — The Entry Point

```python
# Learning mode: Awareness | 22 lines; the simplest function in the file
def main() -> None:
    parser = _build_parser()
    args = parser.parse_args()

    if args.command is None:
        parser.print_help()
        sys.exit(0)

    if args.command == "run":
        _handle_run(args)
    else:
        fn = _STAGE_FN.get(args.command)
        if fn is None:
            parser.error(f"Unknown command: {args.command}")
        result = fn(args)
        if isinstance(result, int):
            sys.exit(result)

if __name__ == "__main__":
    main()
```

**Three branches:**

1. **No command** (`sentinel-data` with no args) → print help, exit 0.
2. **`run`** → delegate to `_handle_run` (which has its own loop).
3. **Anything else** → look up in `_STAGE_FN`, call the handler,
   exit with its return code if it's an int.

**`parser.error()`** (line 1045) — argparse's built-in error handler.
Prints `"error: Unknown command: foo"` and exits with code 2. We
could use `raise SystemExit(2)` but `parser.error` is idiomatic.

**`if __name__ == "__main__"`** (line 1052) — allows `python -m
sentinel_data.cli` or `python cli.py` to work. The `pyproject.toml`
entry point (`sentinel-data = "sentinel_data.cli:main"`) calls
`main()` directly.

---

## §6 The Full Dispatch Flow (ASCII)

```
User types: sentinel-data represent --source solidifi --limit 100

main()
  → parser.parse_args()
  → args.command = "represent"
  → fn = _STAGE_FN["represent"]  →  _run_represent
  → result = _run_represent(args)
  → args.source = "solidifi", args.limit = 100
  → lazy import: from sentinel_data.representation.orchestrator import represent_source
  → cfg = _load_config(args.config)
  → represent_source("solidifi", cfg, data_dir, limit=100, ...)
  → print stats
  → return None
  → main() exits normally (code 0)
```

vs.

```
User types: sentinel-data run --from-stage represent --dry-run

main()
  → args.command = "run"
  → _handle_run(args)
  → start_idx = STAGES.index("represent")  →  2
  → stages_to_run = ["represent", "label", "verify", "split", "register", "analyze", "export"]
  → dry_run = True
  → print "[run --dry-run] Would execute 7 stage(s):"
  → print each stage name + description
  → return
  → main() exits normally (code 0)
```

---

## §7 3 Things to Lock In (per global P10-C)

1. **`_STAGE_FN` is the CLI's dispatch table; `STAGES` is the
   pipeline's stage list.** They overlap but serve different purposes.
   `_STAGE_FN` includes `freshness` (a utility); `STAGES` doesn't.
   Adding a stage requires updating both.

2. **`_handle_run` constructs a default Namespace for each stage.**
   Per-stage CLI args are unavailable in `run` mode. This is a
   deliberate design choice: `run` mode uses defaults; per-stage
   mode lets you customize. The trade-off is explicit.

3. **`main()` has exactly 3 branches: no-command (help), `run`
   (orchestrator), and everything else (single-stage dispatch).**
   The `isinstance(result, int)` check at line 1048 is how exit
   codes propagate from handlers to the OS.

---

## §8 Abbreviations introduced this session (per global P12)

- **OS** — Operating System (the process that runs the CLI)
- **O(n)** — Big-O notation for time complexity; here, linear scan of a 9-element list

---

## §9 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> If you add a new stage called `lint` to the pipeline, list every
> place in `cli.py` that must be updated for `sentinel-data lint`
> to work. (This is the same shape as Session 02's Q4 but now you
> know the full file.)

### Q2 [Pattern]

> `_handle_run` passes `dry_run=False` to every stage handler (line
> 1012), even when the user passed `--dry-run` to `run`. Why? What
> would happen if it passed `args.dry_run` instead?

### Q3 [Portable🔵]

> The `run` subparser uses `choices=STAGES` for `--from-stage`,
> which rejects invalid stage names at parse time. But `_STAGE_FN`
> does a dict lookup (`_STAGE_FN.get(args.command)`) which returns
> `None` for unknown commands. Why does the CLI use two different
> validation strategies (argparse `choices` vs dict lookup), and
> what would be the downside of using only one?

### Q4 [Mechanism]

> `main()` at line 1048 does `if isinstance(result, int):
> sys.exit(result)`. If a future handler returns `True` (a bool)
> instead of `1` (an int), what happens? Does the pipeline abort?
> Why or why not?

---

## §10 Connections (what to study next)

- **Session 05 — Ingestion overview:** `sentinel_data/ingestion/ingest.py`
  + `manifest.py` + `freshness.py`. Now that the full CLI is covered,
  we move into the first subpackage. Session 05 reads the Stage 1a
  orchestrator (ingest.py), the SHA-256 manifest (manifest.py),
  and the freshness utility (freshness.py) that Session 04's
  `_run_freshness` dispatches to.
- **Session 06 — Ingestion connectors:** `connectors/base.py` +
  `git_connector.py` + `manual_connector.py`. The strategy pattern
  behind Stage 1a's multi-source ingestion.

The roadmap in `reference.md §9` has the full 36-session plan.

---

## §11 What "got it" looks like for Session 04

- [ ] Explain the difference between `STAGES` and `_STAGE_FN` and
      why both exist.
- [ ] Trace the dispatch of `sentinel-data run --from-stage
      represent --dry-run` from `main()` through `_handle_run`.
- [ ] Explain why `_handle_run` constructs a default Namespace
      instead of passing `args` directly to each handler.
- [ ] List every place in `cli.py` you'd touch to add a 10th stage.
- [ ] Predict what happens if a handler returns `True` instead of
      `1` (Q4).
- [ ] Explain the 3 branches in `main()` and why each exists.

---

**Next:** answer Q1–Q4 in chat (or paste them below). I'll gap-fill
(P2), update the **Your answers** sections, and update
`session_log.md` for this session. After Q&A, we move to Session 05
(ingestion overview).
