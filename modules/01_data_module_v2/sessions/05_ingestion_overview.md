# Session 05 — Ingestion Overview (`ingestion/ingest.py` + `manifest.py` + `freshness.py`)

**Date:** 2026-06-14
**Session number:** 05 of 36
**Mode:** Understand (first subpackage deep-dive; strategy pattern preview for Session 06)
**Estimated study time:** 1 hr teaching + 15 min Q&A
**Status:** ✅ Completed

---

## §0 Recap of Session 04 + Model Answers to Q1–Q4

### §0.1 Recap of Session 04

Session 04 closed out `cli.py` (lines 700-1053). Key concepts:
**(1)** `_STAGE_FN` is the dispatch dict (CLI commands); `STAGES` is the pipeline
stage list — they overlap but serve different purposes. **(2)** `_handle_run`
constructs a default Namespace for each stage — `run` mode can't customize
per-stage args. **(3)** `main()` has 3 branches: no-command (help), `run`
(orchestrator), else (single-stage dispatch). **(4)** `_build_parser` is
~220 LOC, the largest function in `cli.py` — 11 subparsers with conditional
args per stage.

### §0.2 Model answer to Q1 [Mechanism]

> **Q:** If you add a new stage called `lint` to the pipeline, list every
> place in `cli.py` that must be updated for `sentinel-data lint` to work.

**Model answer (8 places in `cli.py` + 2 outside):**

Inside `cli.py`:

1. **`STAGES` list** (line 71-81) — add `"lint"` at the correct position.
2. **`STAGE_DESCRIPTIONS` dict** (line 83-93) — add a one-line description.
3. **Define a `_run_lint` function** following the 5-step template.
4. **`_STAGE_FN` dispatch table** (line 761-772) — add `"lint": _run_lint`.
5. **`_build_parser`** (line 804-982) — add a new `subparsers.add_parser("lint", ...)` block
   inside the `for stage in STAGES:` loop (it'll be auto-added by the loop, but if
   `lint` has unique args, add a conditional block).
6. **`_handle_run`** (line 1011-1018) — add any new default args to the synthetic
   `stage_args = argparse.Namespace(...)`.
7. **Module docstring** (line 23-33) — add `lint` to the stages list.
8. **`__all__`** — if defined, add the new function. (Not currently defined.)

Outside `cli.py`:

9. **`dvc.yaml`** — add a new `stages: lint:` block.
10. **`README.md`** — update stage status table and pipeline DAG.

### §0.3 Model answer to Q2 [Pattern]

> **Q:** `_handle_run` passes `dry_run=False` to every stage handler, even
> when the user passed `--dry-run` to `run`. Why?

**Model answer:**

Because the `run` subparser's dry-run has **different semantics** than a
per-stage dry-run:

- **`run --dry-run`** prints the list of stages that *would* run and returns.
  It does NOT call any handlers. It's a high-level plan view.
- **Per-stage `--dry-run`** calls the handler, which prints what files it
  would process and returns without writing.

If `_handle_run` passed `args.dry_run=True` to each handler, every
handler would print its own dry-run output *and* `_handle_run` would
also print the stage list — duplicate/confusing output.

The current design: `_handle_run --dry-run` = list of stages;
per-stage `--dry-run` = per-file plan. They're mutually exclusive
entry points.

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** The `run` subparser uses `choices=STAGES` for `--from-stage`
> (rejects invalid names at parse time), but `_STAGE_FN` uses a dict
> lookup (returns `None` for unknown commands). Why two strategies?

**Model answer:**

They serve different validation purposes:

- **`choices=STAGES`** in argparse: validates at **parse time**,
  before any code runs. The user gets a clean error: `"error: invalid
  choice: 'lint' (choose from 'ingest', ...)"`. This is input
  validation — "is this a valid stage name?"

- **`_STAGE_FN.get()`** in `main()`: validates at **dispatch time**,
  after parsing. This is a safety net — "did we remember to register
  this command?" In practice, this path is unreachable (argparse
  already rejected unknown subcommands), but it's defensive
  programming.

**Using only one:**
- Only `choices`: the `main()` dict lookup would never fire for
  invalid commands (argparse catches them first). But if someone
  bypasses argparse (e.g. calls `_handle_run` directly), there's
  no safety net.
- Only dict lookup: the user gets an unformatted traceback instead
  of argparse's clean error message. Worse UX.

The current design: argparse for UX, dict for safety. Belt and
suspenders.

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** `main()` does `if isinstance(result, int): sys.exit(result)`.
> If a future handler returns `True` instead of `1`, what happens?

**Model answer:**

`True` is a `bool`, which is a subclass of `int` in Python:

```python
>>> isinstance(True, int)
True
>>> True == 1
True
```

So `isinstance(True, int)` is `True`, and `sys.exit(True)` is called.
`sys.exit(True)` exits with code 1 (because `bool.__int__(True) == 1`).
So yes, the pipeline would abort with exit code 1 — the same as
returning `1`.

**But this is fragile.** If someone returns `0` (the int) vs `False`
(the bool), the behavior differs:
- `sys.exit(0)` → exit code 0 (success)
- `sys.exit(False)` → `False.__int__()` is `0` → exit code 0 (same)

So actually it works for both `True`/`1` and `False`/`0`. But
relying on `bool` being an `int` subclass is a Python implementation
detail that could confuse readers. The convention should be: return
`int`, not `bool`.

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** The first failed source aborts `ingest_all`. `ConnectorError` propagates uncaught through the for loop in `ingest_all` (no try/except per-source) up to `_run_ingest`, which catches it and prints a user-friendly message. Remaining enabled sources are skipped — this is fail-fast on ingestion. If SolidiFI fails, DIVE and Clear are never attempted.

**Your answer to Q2:** `pin` records the configured intent (e.g., `""` = "use HEAD", `"abc123"` = explicit commit). `resolved_pin` records what was actually fetched. The pair distinguishes explicit pinning from HEAD-at-ingest-time. If `pin=""` and `resolved="abc123"`, I know the source was at HEAD when ingested. If `pin="abc123"` but `resolved="def456"`, the pin is stale. Without both, you lose the audit trail.

**Your answer to Q3:** `importlib.metadata.version()` queries the **current interpreter** without spawning a subprocess. `pip show` via subprocess would run whatever `pip` is on PATH — potentially a different venv. The data venv and ml venv may have different Slither versions. Principle: when checking the current runtime's packages, use the runtime's own metadata API, not a subprocess that might invoke a different interpreter.

**Your answer to Q4:** `_source_config()` collects non-standard config keys into `SourceConfig.extra` (lines 41-46). The `source_cfg` is passed to `connector.pull(source_cfg, raw_dir)` (line 78 of `ingest.py`). `BaseConnector.pull()` delegates to `self._pull(cfg, dest)` (line 55 of `base.py`). The connector's `_pull()` reads `cfg.extra` — e.g., `ManualConnector._pull` does `extra = cfg.extra or {}` then `extra.get("staging_path")`. The `extra` dict is the type-safe bypass: source-specific keys flow through without modifying the `SourceConfig` schema.

---

## §1 What This Chunk Covers (per global P5)

This is the **first subpackage deep-dive**: `sentinel_data/ingestion/`,
Stage 1a of the pipeline. Three files:

| File | LOC | Role |
|------|-----|------|
| `ingest.py` | 113 | **Orchestrator** — reads config, picks connector, calls `pull()`, writes manifest |
| `manifest.py` | 111 | `IngestionManifest` + `FileRecord` + SHA-256 verification + append-only semantics |
| `freshness.py` | 120 | Pin staleness checker (git HEAD + Slither version) → `freshness_report.md` |

**Why this order in the pipeline:** ingestion is the front door. Every
contract that eventually becomes part of the training dataset first
passes through this module. The BCCC corpus (predecessor) had 38.8%
duplication across sources and no way to reconstruct which source
version was used. The manifest is the antidote.

---

## §2 `ingest.py` — The Orchestrator

### §2.1 `_all_sources` + `_enabled_sources` (lines 15-26)

```python
# Learning mode: Understand | Merges 3 config sections, filters to enabled
def _all_sources(cfg: dict) -> dict[str, dict]:
    """Merge sources_critical_path + sources_additive into one flat dict."""
    out: dict = {}
    out.update(cfg.get("sources_critical_path") or {})
    out.update(cfg.get("sources_additive") or {})
    out.update(cfg.get("sources") or {})        # legacy v1 key
    return out

def _enabled_sources(cfg: dict) -> dict[str, dict]:
    """Filter to only sources with ``enabled: true`` in config."""
    return {k: v for k, v in _all_sources(cfg).items() if v.get("enabled")}
```

**Three config sections merged:**
1. `sources_critical_path` — v2 sources that Run 11/12 ships with
2. `sources_additive` — v2.1 deferred sources
3. `sources` — legacy v1 key (kept for back-compat)

**The `or {}` idiom:** if the key is missing or `None`, default to
empty dict. Prevents `AttributeError` on `.get()`.

**Why merge order matters:** if a source name appears in multiple
sections, the later `update()` overwrites the earlier one. The legacy
`sources` section is last, so it can override v2 entries. This is
intentional — it lets you patch a v2 source config via the legacy
key without modifying the v2 section.

### §2.2 `_source_config` (lines 29-46)

```python
# Learning mode: Understand | Converts raw config dict to typed SourceConfig dataclass
def _source_config(name: str, entry: dict) -> SourceConfig:
    return SourceConfig(
        name=name,
        connector=entry.get("connector", "git"),
        url=entry.get("url", ""),
        pin=entry.get("pin", ""),
        hf_dataset=entry.get("hf_dataset", ""),
        zenodo_record=entry.get("zenodo_record", ""),
        description=entry.get("description", ""),
        include_subdirs=list(entry.get("include_subdirs") or []),
        exclude_subdirs=list(entry.get("exclude_subdirs") or []),
        extra={k: v for k, v in entry.items()
               if k not in ("enabled", "tier", "tier_subtype", "connector", ...)},
    )
```

**The `extra` dict** (lines 41-46) catches everything that isn't a
standard field. This is how source-specific config (like DIVE's
`staging_path`, `labels_csv`, `dive_class_mapping`) flows through
to the connector without polluting the typed interface.

**The `list(... or [])` idiom** (lines 39-40): if `include_subdirs`
is `None` in config, default to empty list. `list()` ensures we get
a new list (not a reference to the config's list).

### §2.3 `ingest_source` (lines 49-92) — The 5-Step Call Chain

```python
# Learning mode: Understand | The core orchestration function
def ingest_source(name, cfg, data_dir, dry_run=False) -> IngestionManifest | None:
    sources = _enabled_sources(cfg)                    # Step 1: resolve source
    if name not in sources:
        ...raise ConnectorError...
    entry = sources[name]
    source_cfg = _source_config(name, entry)           # Step 2: build SourceConfig
    raw_dir = data_dir / "raw" / name                  # Step 3: resolve dest

    if dry_run:
        print(...)                                     # Step 3a: dry-run output
        return None

    connector = get_connector(source_cfg.connector)    # Step 4: get connector
    result = connector.pull(source_cfg, raw_dir)       # Step 4a: pull!

    manifest = IngestionManifest(                      # Step 5: build + save manifest
        source=name, connector=source_cfg.connector, ...,
        files=build_file_records(result.sol_files, raw_dir),
    )
    manifest.save(raw_dir / "ingestion_manifest.json")
    return manifest
```

**The error handling** (lines 61-65): two distinct error cases:
1. Source exists in config but `enabled: false` → "exists but is not enabled"
2. Source not in config at all → "not found in config.yaml"

Both raise `ConnectorError`, which the CLI handler catches and
prints as a user-friendly error.

**The `connector.pull()` call** (line 78): this is where the
strategy pattern kicks in. `get_connector()` returns a
`GitConnector`, `ManualConnector`, etc. The orchestrator doesn't
know which one — it just calls `.pull()`. Session 06 covers
the connectors in detail.

**`build_file_records`** (line 89): computes SHA-256 of every `.sol`
file. This is the provenance trail — the manifest records exactly
which files were ingested and their hashes.

### §2.4 `ingest_all` (lines 95-113)

```python
# Learning mode: Awareness | Iterates enabled sources, calls ingest_source for each
def ingest_all(cfg, data_dir, dry_run=False) -> list[IngestionManifest]:
    sources = _enabled_sources(cfg)
    manifests = []
    for name in sources:
        print(f"\n[ingest] {name}")
        m = ingest_source(name, cfg, data_dir, dry_run=dry_run)
        if m:
            manifests.append(m)
            print(f"  contracts   : {m.contract_count}")
            print(f"  resolved    : {m.resolved_pin[:12]}")
    return manifests
```

**`resolved_pin[:12]`** — truncates the 40-char commit SHA to 12
chars for display. Enough to identify the commit, short enough
to not clutter output.

**Return type:** `list[IngestionManifest]` — one per successfully
ingested source. Failed sources raise `ConnectorError` (not caught
here; the CLI handler catches it).

---

## §3 `manifest.py` — The Audit Trail

### §3.1 `FileRecord` (lines 18-23)

```python
# Learning mode: Understand | Per-file SHA-256 fingerprint
@dataclass
class FileRecord:
    path: str           # relative to the source raw dir
    sha256: str
    size_bytes: int
```

**`path` is relative** to the raw dir (e.g. `repo/buggy_contracts/reentrancy.sol`).
This makes the manifest portable — if you move the raw dir, the
manifest still works as long as the internal structure is preserved.

### §3.2 `IngestionManifest` (lines 26-77)

```python
# Learning mode: Master | Append-only; re-ingest re-validates every SHA-256
@dataclass
class IngestionManifest:
    source: str
    connector: str
    url: str
    pin: str                           # configured pin (may be empty)
    resolved_pin: str                  # actual commit/hash at fetch time
    fetched_at: str                    # ISO-8601
    duration_s: float
    contract_count: int
    files: list[FileRecord] = field(default_factory=list)
    extra: dict[str, Any] = field(default_factory=dict)
```

**`pin` vs `resolved_pin`:**
- `pin` = what's in `config.yaml` (may be empty = "use HEAD")
- `resolved_pin` = what the connector actually resolved at fetch
  time (the git commit SHA, the Zenodo record version, etc.)

**This distinction matters for reproducibility.** If `pin=""` and
`resolved_pin="abc123..."`, you know the source was at HEAD when
you ingested. If `pin="abc123..."` and `resolved_pin="abc123..."`,
the pin is confirmed.

### §3.3 `verify` (lines 63-77)

```python
# Learning mode: MASTER | Re-checks every SHA-256; fails loud on any mismatch
def verify(self, raw_dir: Path) -> tuple[bool, list[str]]:
    errors: list[str] = []
    for rec in self.files:
        fpath = raw_dir / rec.path
        if not fpath.exists():
            errors.append(f"MISSING  {rec.path}")
            continue
        actual = _sha256(fpath)
        if actual != rec.sha256:
            errors.append(f"CHANGED  {rec.path}  expected=... got=...")
    return (len(errors) == 0), errors
```

**The verification contract:**
- `MISSING` = file was deleted (disk corruption, accidental `rm`)
- `CHANGED` = file content differs from what was ingested (upstream
  force-push, accidental edit, disk corruption)

**When this runs:** `verify_manifest()` at line 108-111 is called
by Stage 1b (preprocessing) as a load-time gate. If any file
changed, preprocessing fails before doing any work.

> **AUDIT: A#001** — The manifest is verified at preprocessing time
> but NOT at representation time. If a file is corrupted between
> preprocessing and representation, the `.pt` graph will be built
> from corrupt data. A verify call at representation entry would
> catch this.

### §3.4 `_sha256` (lines 82-88)

```python
# Learning mode: Understand | 64 KiB chunked read for memory efficiency
def _sha256(path: Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            h.update(chunk)
    return h.hexdigest()
```

**Why chunked:** Solidity files are small (typically <100 KB), but
some corpus files (DISL) can be larger. Reading 64 KiB at a time
keeps memory constant regardless of file size. The `iter(lambda:
f.read(65536), b"")` pattern is a standard Python idiom for
reading until EOF.

### §3.5 `build_file_records` (lines 91-100)

```python
# Learning mode: Awareness | Creates FileRecord entries with SHA-256
def build_file_records(sol_files: list[Path], base_dir: Path) -> list[FileRecord]:
    records = []
    for p in sol_files:
        records.append(FileRecord(
            path=str(p.relative_to(base_dir)),
            sha256=_sha256(p),
            size_bytes=p.stat().st_size,
        ))
    return records
```

Called by `ingest_source` after the connector pulls. The `sol_files`
list comes from `PullResult.sol_files` — the connector already
discovered which `.sol` files exist. This function just records
their hashes.

---

## §4 `freshness.py` — The Early-Warning System

### §4.1 `run_freshness_check` (lines 17-62)

```python
# Learning mode: Understand | Informational report, not blocking
def run_freshness_check(cfg: dict, data_dir: Path) -> str:
    lines = ["# Freshness Report", f"Generated: {datetime.utcnow().isoformat()}Z", ""]

    from sentinel_data.ingestion.ingest import _all_sources
    for name, entry in _all_sources(cfg).items():
        if not entry.get("enabled"):
            continue
        pin = entry.get("pin", "")
        url = entry.get("url", "")
        connector = entry.get("connector", "git")

        if connector == "git" and url:
            upstream = _git_upstream_head(url)
            if upstream and pin and not upstream.startswith(pin):
                status = f"STALE — pinned={pin[:12]} upstream={upstream[:12]}"
            elif not pin:
                status = f"UNPINNED — upstream HEAD=..."
            else:
                status = "OK"
        else:
            status = f"UNCHECKED (connector={connector})"

        lines.append(f"- **{name}**: {status}")

    lines += ["", "## Slither Version", ""]
    lines.append(_slither_version_check())
    ...write to data/analysis/freshness_report.md...
```

**The freshness check is informational, not blocking.** It doesn't
prevent the pipeline from running. It produces a report for human
review.

**Three possible statuses per source:**
1. **OK** — pin matches upstream HEAD
2. **STALE** — upstream has moved past the pin
3. **UNPINNED** — no pin configured (tracks HEAD implicitly)
4. **UNCHECKED** — non-git connector (manual, HuggingFace, etc.)

### §4.2 `_git_upstream_head` (lines 65-76)

```python
# Learning mode: Understand | `git ls-remote` — no clone needed
def _git_upstream_head(url: str) -> str:
    try:
        result = subprocess.run(
            ["git", "ls-remote", url, "HEAD"],
            capture_output=True, text=True, timeout=15,
        )
        if result.returncode == 0 and result.stdout:
            return result.stdout.split()[0]
    except Exception:
        pass
    return ""
```

**`git ls-remote`** queries the remote without cloning. It returns
the SHA of `HEAD` (and other refs). The `timeout=15` prevents
hanging on unreachable repos.

**Why `split()[0]`:** `git ls-remote` output is `<sha>\t<ref>`.
We want just the sha.

### §4.3 `_slither_version_check` (lines 79-93)

```python
# Learning mode: MASTER🔵 | The Run 9 Slither breakage detector
def _slither_version_check() -> str:
    installed = _installed_slither_version()
    latest = _latest_pypi_version("slither-analyzer")
    if not installed:
        return "slither-analyzer: NOT INSTALLED ..."
    if not latest:
        return f"slither-analyzer: installed={installed} | could not check PyPI"
    if installed == latest:
        return f"slither-analyzer: OK ({installed})"
    return (
        f"slither-analyzer: STALE — installed={installed} latest={latest}. "
        "API changes between versions can silently break graph_extractor.py "
        "(see ADR-0002)."
    )
```

> 🔵 **Portable recall — Slither API drift:** Slither is a static
> analysis tool for Solidity. Its Python API changes between
> versions. In Run 9, a Slither API change silently broke
> `graph_extractor.py`, producing corrupt graphs. The freshness
> check is the early warning: if the installed Slither doesn't
> match the latest PyPI release, flag it for review.

**`_installed_slither_version`** (lines 96-108): uses
`importlib.metadata.version()` scoped to the **current interpreter**.
The docstring explains why: subprocess fallback would read a different
Python's packages, which is wrong when the data venv and ml venv
have different Slither versions.

**`_latest_pypi_version`** (lines 111-120): queries PyPI's JSON
API (`https://pypi.org/pypi/slither-analyzer/json`). The `timeout=5`
prevents hanging on network issues.

---

## §5 The `connectors/` Sub-package (Preview for Session 06)

The `connectors/__init__.py` has a **registry pattern**:

```python
# Learning mode: Awareness | Strategy pattern — 5 types, 2 aliases
_REGISTRY: dict[str, type[BaseConnector]] = {
    "git":          GitConnector,
    "huggingface":  HuggingFaceConnector,
    "zenodo":       ZenodoConnector,
    "etherscan":    EtherscanConnector,
    "manual":       ManualConnector,
    "audit_report": ManualConnector,   # alias
    "rekt_scraper": ManualConnector,   # alias
}

def get_connector(connector_type: str) -> BaseConnector:
    cls = _REGISTRY.get(connector_type)
    if cls is None:
        raise ConnectorError(f"Unknown connector type {connector_type!r}")
    return cls()
```

**The `BaseConnector` interface** (from `connectors/base.py`):

```python
class BaseConnector(ABC):
    def pull(self, cfg: SourceConfig, dest: Path) -> PullResult:
        dest.mkdir(parents=True, exist_ok=True)
        start = time.monotonic()
        result = self._pull(cfg, dest)      # ← subclass implements this
        result.duration_s = time.monotonic() - start
        result.fetched_at = datetime.utcnow().isoformat() + "Z"
        return result

    @abstractmethod
    def _pull(self, cfg: SourceConfig, dest: Path) -> PullResult: ...

    @staticmethod
    def find_sol_files(root, include_subdirs=None, exclude_subdirs=None) -> list[Path]: ...
```

**The strategy pattern:** the orchestrator calls
`get_connector("git")` → gets a `GitConnector` → calls `.pull()`.
The orchestrator doesn't know if it's git, HuggingFace, or manual.
Each connector implements `_pull()` differently.

**`find_sol_files`** — static method shared by all connectors.
Uses `include_subdirs` (allowlist) and `exclude_subdirs` (blocklist)
to filter. SolidiFI uses `include_subdirs=["buggy_contracts"]` to
skip the analysis-tool output in `results/`.

Session 06 covers the 3 active connectors (git, manual, etherscan)
in detail.

---

## §6 The 5-Step Ingest Call Chain (Full Trace)

```
sentinel-data ingest --source solidifi

cli.py:_run_ingest (line 111)
  → lazy import: from sentinel_data.ingestion.ingest import ingest_source
  → cfg = _load_config(args.config)
  → ingest_source("solidifi", cfg, data_dir)

ingest.py:ingest_source (line 49)
  → sources = _enabled_sources(cfg)           # merge 3 config sections, filter
  → source_cfg = _source_config("solidifi", entry)  # dict → SourceConfig dataclass
  → raw_dir = data_dir / "raw" / "solidifi"
  → connector = get_connector("git")          # → GitConnector()
  → result = connector.pull(source_cfg, raw_dir)

connectors/base.py:BaseConnector.pull (line 46)
  → dest.mkdir(parents=True, exist_ok=True)
  → result = self._pull(cfg, dest)            # GitConnector._pull: git clone + checkout
  → result.duration_s = ...
  → result.fetched_at = ...
  → return PullResult(sol_files=[...], resolved_pin="abc123...", ...)

ingest.py:ingest_source (continued)
  → manifest = IngestionManifest(
        source="solidifi", resolved_pin="abc123...",
        files=build_file_records(result.sol_files, raw_dir),  # SHA-256 per file
    )
  → manifest.save(raw_dir / "ingestion_manifest.json")
  → return manifest
```

---

## §7 3 Things to Lock In (per global P10-C)

1. **The manifest is append-only and re-validates every SHA-256
   on re-ingest.** This is the audit trail that lets you answer
   "what exact version of SolidiFI did Run 12 train on?" six months
   from now. The BCCC corpus (predecessor) had no such trail and
   38.8% cross-source duplication.

2. **`_all_sources` merges 3 config sections in order: critical_path
   → additive → legacy.** Later sections override earlier ones. This
   is intentional — the legacy `sources` key can patch v2 entries.

3. **The freshness check is informational, not blocking.** It flags
   stale pins and Slither version mismatches for human review. The
   Slither check exists because API changes silently broke
   `graph_extractor.py` in Run 9 (ADR-0002).

---

## §8 Abbreviations introduced this session (per global P12)

- **SHA-256** — Secure Hash Algorithm 256-bit (cryptographic hash function; produces a 64-char hex string)
- **ISO-8601** — international standard for date/time representation (e.g. `2026-06-14T12:00:00Z`)
- **HEAD** — the latest commit on the default branch of a git repository
- **PyPI** — Python Package Index (the official Python package repository at pypi.org)
- **idempotent** — running the same operation twice produces the same result as running it once

---

## §9 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> `ingest_source` raises `ConnectorError` if the source is not found
> or not enabled (lines 61-65). But `ingest_all` (line 108) does NOT
> catch `ConnectorError`. What happens if one source fails during
> `ingest_all`? Does the entire ingestion abort?

### Q2 [Pattern]

> The manifest stores `pin` (what's in config) and `resolved_pin`
> (what was actually fetched). Why not just store `resolved_pin`?
> What's the value of recording the original pin alongside the
> resolved one?

### Q3 [Portable🔵]

> `freshness.py` uses `importlib.metadata.version("slither-analyzer")`
> to check the installed version, and PyPI's JSON API for the latest
> version. Why not just run `pip show slither-analyzer` via subprocess?
> What's the principle of checking versions programmatically vs.
> shelling out?

### Q4 [Mechanism]

> `_source_config` builds an `extra` dict (lines 41-46) that catches
> all non-standard keys from the config entry. For DIVE, this includes
> `staging_path`, `labels_csv`, `dive_class_mapping`, etc. How does
> this `extra` dict flow through to the connector? Trace the path
> from `ingest.py:_source_config` → `SourceConfig.extra` → connector
> `_pull()`.

---

## §10 Connections (what to study next)

- **Session 06 — Ingestion connectors:** `connectors/base.py` +
  `git_connector.py` + `manual_connector.py`. The 3 active connector
  implementations. You'll see how `git clone + checkout` works for
  git sources, and how ZIP extraction + macOS `__MACOSX/` stripping
  works for manual sources.
- **Session 07 — Label folderize:** `label_folderize.py` (173 LOC).
  The DIVE-specific symlink logic that creates per-class folders.

---

## §11 What "got it" looks like for Session 05

- [ ] Trace the 5-step ingest call chain from CLI to manifest
      without looking at the code.
- [ ] Explain `pin` vs `resolved_pin` and why both are stored.
- [ ] Describe the manifest verification contract: what does
      `MISSING` vs `CHANGED` mean, and when does verification run?
- [ ] Explain why the freshness check is informational (not
      blocking) and what would change if it were made blocking.
- [ ] Name the 3 config sections that `_all_sources` merges and
      explain the merge order's significance.
- [ ] Describe the strategy pattern in `connectors/` and how
      `get_connector()` dispatches.

---

**Next:** answer Q1–Q4 in chat. I'll gap-fill (P2), update
**Your answers**, and update `session_log.md`. After Q&A, we move
to Session 06 (connectors).
