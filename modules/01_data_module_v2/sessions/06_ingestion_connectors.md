# Session 06 — Ingestion Connectors (`connectors/base.py` + `git_connector.py` + `manual_connector.py`)

**Date:** 2026-06-14
**Session number:** 06 of 36
**Mode:** Understand (strategy pattern deep-dive; first concrete connector implementations)
**Estimated study time:** 1 hr teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0 Recap of Session 05 + Model Answers to Q1–Q4

### §0.1 Recap of Session 05

Session 05 covered `ingestion/ingest.py` (113 LOC), `manifest.py` (111 LOC), and `freshness.py` (120 LOC). Key concepts: **(1)** `_all_sources` merges 3 config sections (critical_path → additive → legacy) in override order. **(2)** `ingest_source` has a 5-step call chain: resolve source → build SourceConfig → resolve dest → get connector → pull → build + save manifest. **(3)** The manifest stores both `pin` (configured intent) and `resolved_pin` (actual fetched commit). **(4)** Freshness check is informational, not blocking — flags stale pins and Slither version mismatches.

### §0.2 Model answer to Q1 [Mechanism]

> **Q:** `ingest_source` raises `ConnectorError` if the source is not found or not enabled. But `ingest_all` does NOT catch `ConnectorError`. What happens if one source fails during `ingest_all`? Does the entire ingestion abort?

**Model answer:** Yes, the entire ingestion aborts. The first failed source propagates `ConnectorError` uncaught through `ingest_all`'s for loop (no try/except per-source) up to `_run_ingest`, which catches it and prints `f"  Error: {e}"`. Remaining enabled sources are never attempted.

This is **fail-fast on ingestion**. The rationale: if a critical-path source fails, later stages (preprocess, represent) would fail anyway because the manifest is incomplete. Fail-fast > silent-skip. However, there's no continuation mode — if source 3 of 10 fails, sources 4-10 are lost in that run.

### §0.3 Model answer to Q2 [Pattern]

> **Q:** The manifest stores `pin` (what's in config) and `resolved_pin` (what was actually fetched). Why not just store `resolved_pin`? What's the value of recording the original pin alongside the resolved one?

**Model answer:** The pair answers two different questions:
- `pin` = "what was the source *supposed* to be?" (configured intent; `""` means "track HEAD")
- `resolved_pin` = "what did we *actually* get?" (the git SHA at ingest time)

Without both, the audit trail is ambiguous:
- Only `resolved_pin="abc123"` → was it pinned or HEAD-at-time? Unknown.
- Only `pin="abc123"` → did the fetch actually resolve to this? Unknown.

The pair also enables staleness detection: if `pin` is non-empty and `pin != resolved_pin`, the pin didn't resolve. If `pin=""` and upstream HEAD has moved past `resolved_pin`, the source is stale (freshness check).

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** `freshness.py` uses `importlib.metadata.version("slither-analyzer")` to check the installed version, and PyPI's JSON API for the latest version. Why not just run `pip show slither-analyzer` via subprocess? What's the principle?

**Model answer:** `importlib.metadata.version()` queries the **current Python process's** metadata — the exact interpreter running the freshness check. `pip show` via subprocess runs whatever `pip` is on PATH, which may belong to a different venv (e.g., the system Python's pip instead of the data venv's pip). The data venv and ml venv have different Slither versions — querying the wrong one gives a false freshness report.

**Principle:** When a program needs information about its own runtime, use introspection APIs (`importlib.metadata`, `sys.version`, etc.) rather than shelling out to external tools. Shelling out couples the program to its execution environment (PATH, env vars, filesystem) in ways that introspection APIs avoid.

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** `_source_config` builds an `extra` dict that catches all non-standard keys. For DIVE, this includes `staging_path`, `labels_csv`, `dive_class_mapping`, etc. How does this `extra` dict flow through to the connector? Trace the path from `ingest.py:_source_config` → `SourceConfig.extra` → connector `_pull()`.

**Model answer:** The flow has 5 hops:

1. **`_source_config()`** in `ingest.py` (lines 41-46): collects all config entry keys not in the standard set → `SourceConfig.extra`.

2. **`SourceConfig.extra`** in `base.py` (line 24): `dict[str, Any]` dataclass field — a typed catch-all.

3. **`connector.pull(source_cfg, raw_dir)`** in `ingest_source` (line 78): passes the entire `SourceConfig` with `extra` intact.

4. **`BaseConnector.pull()`** in `base.py` (line 55): calls `self._pull(cfg, dest)` — `cfg` is the same `SourceConfig`.

5. **Connector `_pull()`** reads `cfg.extra`:
   - `manual_connector.py` line 56: `extra = cfg.extra or {}` then `extra.get("staging_path")`
   - `git_connector.py` line 38: `cfg.extra.get("post_clone_cmd")`

**Principle:** `extra` is a type-safe bypass. Source-specific config keys (DIVE's `staging_path`, scabench's `post_clone_cmd`) flow through without modifying the `SourceConfig` schema or the connector interface.

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** _pending_

**Your answer to Q2:** _pending_

**Your answer to Q3:** _pending_

**Your answer to Q4:** _pending_

---

## §1 What This Chunk Covers (per global P5)

| File | LOC | Role |
|------|-----|------|
| `base.py` | 95 | `BaseConnector` ABC + `SourceConfig` + `PullResult` + `find_sol_files()` |
| `__init__.py` | 38 | Registry dict `_REGISTRY` + `get_connector()` factory |
| `git_connector.py` | 89 | Git clone (full/shallow) + pin checkout + post-clone hooks |
| `manual_connector.py` | 189 | Staging materialization (zip/dir/glob) + macOS cleanup |

**Why this module exists (P0):** Ingestion is the front door. Every `.sol` file in the dataset enters through one of 5 connectors (2 implemented, 3 stubs). The strategy pattern keeps the orchestrator (`ingest.py`) agnostic to *which* connector is used — it calls `.pull()` and gets a `PullResult` back regardless.

**Connector implementation status:**
- `git`: ✅ **89 LOC, fully implemented** — clone + pin checkout + `post_clone_cmd`
- `manual`: ✅ **189 LOC, fully implemented** — zip/dir/glob materialization
- `etherscan`: ❌ **STUB** — raises `NotImplementedError` (Stage 1 for disl, 514K unlabeled contracts)
- `huggingface`: ❌ **STUB** — raises `NotImplementedError` (Stage 1 for `slither_audited` / `solidity_defi_vulns`)
- `zenodo`: ❌ **STUB** — raises `NotImplementedError` (Stage 1 for `zenodo_clear`, record 16910242)

This session covers only the 2 implemented connectors. The 3 stubs follow the same interface pattern — they just lack the `_pull` logic.

---

## §2 `base.py` — The Interface (95 LOC)

### §2.1 `SourceConfig` (lines 12-24)

```python
@dataclass
class SourceConfig:
    name: str
    connector: str
    url: str = ""
    pin: str = ""
    hf_dataset: str = ""
    ...
    extra: dict[str, Any] = field(default_factory=dict)
```

`SourceConfig` is the **adapter** between the raw `config.yaml` dict and the typed connector interface. The `_source_config()` function in `ingest.py` converts dict → `SourceConfig`. Connectors never touch the raw config — they receive a typed object with known fields.

### §2.2 `PullResult` (lines 27-36)

```python
@dataclass
class PullResult:
    source: str
    local_dir: Path
    resolved_pin: str
    sol_files: list[Path]
    fetched_at: str
    duration_s: float
    extra: dict[str, Any] = field(default_factory=dict)
```

**The return contract** — every connector returns the same shape:
- `local_dir`: where the source data lives on disk
- `resolved_pin`: what was actually fetched (git SHA, mtime, version string)
- `sol_files`: discovered `.sol` files (relative or absolute paths)

**`fetch_at` and `duration_s` are filled by `BaseConnector.pull()`**, not by the subclass's `_pull()`. The template method pattern:

```python
# Learning mode: Understand | Template method — timing + mkdir is always the same
def pull(self, cfg: SourceConfig, dest: Path) -> PullResult:
    dest.mkdir(parents=True, exist_ok=True)
    start = time.monotonic()
    result = self._pull(cfg, dest)            # ← subclass implements
    result.duration_s = time.monotonic() - start
    result.fetched_at = datetime.utcnow().isoformat() + "Z"
    return result
```

The ABC calls `_pull()` (the subclass hook) and wraps it with timing + directory creation. Every connector gets timing for free.

### §2.3 `ConnectorError` (line 39-40)

```python
class ConnectorError(Exception):
    """Raised when a connector cannot complete a pull."""
```

Used across all connectors for failures: missing URL, clone failure, bad zip, glob mismatches.

### §2.4 `find_sol_files` — The Shared Static Method (lines 64-95)

```python
# Learning mode: Master | Allowlist via include_subdirs, blocklist via exclude_subdirs
@staticmethod
def find_sol_files(root: Path, include_subdirs=None, exclude_subdirs=None) -> list[Path]:
    if include_subdirs:
        roots = [root / s for s in include_subdirs]      # only search these subdirs
    else:
        roots = [root]                                   # search the whole root
    ...
    for p in r.rglob("*.sol"):
        if exclude_subdirs:
            rel = p.relative_to(root)
            top = rel.parts[0]
            if top in exclude_subdirs:
                continue
        out.append(p)
    return sorted(out)
```

**Allowlist vs blocklist semantics:**
- `include_subdirs` (allowlist): limits the **search space** to specific top-level subdirs. If empty, search the whole root.
- `exclude_subdirs` (blocklist): filters **after** the glob. Files whose top-level subdir is in the blocklist are skipped.

**Interaction**: If a subdir is in both lists, it's included (entered in the search) then excluded (filtered after glob). `exclude_subdirs` wins — but this is redundant; don't put the same dir in both.

**Why this matters**: SolidiFI has `include_subdirs=["buggy_contracts"]` to skip analysis-tool output in `results/`. Without the allowlist, `find_sol_files` would crawl Mythril/Slither/Smartcheck JSON outputs looking for `.sol` files (it wouldn't find any, but the crawl is wasteful).

**`sorted(out)`** returns a deterministic order regardless of filesystem ordering — important for SHA-256 manifest reproducibility.

---

## §3 `__init__.py` — The Registry Pattern (38 LOC)

```python
# Learning mode: Understand | Strategy pattern via dict lookup
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
        raise ConnectorError(
            f"Unknown connector type {connector_type!r}. "
            f"Available: {sorted(_REGISTRY)}"
        )
    return cls()
```

**7 entries → 5 unique connectors** (3 types + 2 aliases for ManualConnector):
- `git`: the only one with full git operations
- `manual`, `audit_report`, `rekt_scraper`: all same class, different config keys
- `huggingface`, `zenodo`, `etherscan`: 3 more connector specializations

**Factory pattern**: `get_connector("git")` returns a new `GitConnector()` instance. The orchestrator calls `.pull()` on the instance. No connector state is shared between calls.

**Why aliases exist**: `audit_report` and `rekt_scraper` are semantically different source types but use the same materialization logic as "manual download". The aliases make `config.yaml` self-documenting — you write `connector: audit_report` rather than `connector: manual` when the source is a manually-downloaded audit report.

---

## §4 `git_connector.py` — Git Cloning (89 LOC)

### §4.1 `_pull` — The Entry Point (lines 25-55)

```python
# Learning mode: Understand | 3 clauses: no-pin shallow, pin full, existing re-use
def _pull(self, cfg: SourceConfig, dest: Path) -> PullResult:
    repo_dir = dest / "repo"

    if repo_dir.exists():
        resolved = self._current_commit(repo_dir)          # Case 1: re-use
    else:
        resolved = self._clone(cfg, repo_dir)              # Case 2: clone

    # post-clone command (e.g. `python checkout_sources.py` for scabench)
    post_cmd = cfg.extra.get("post_clone_cmd")
    if post_cmd and not (dest / ".post_clone_done").exists():
        _run(post_cmd.split(), cwd=repo_dir)
        (dest / ".post_clone_done").touch()

    sol_files = self.find_sol_files(...)
    return PullResult(source=cfg.name, local_dir=repo_dir, ...)
```

**3 paths through `_pull`:**
1. `repo_dir` already exists → re-use (idempotent re-run). Returns current HEAD SHA.
2. `repo_dir` doesn't exist, `cfg.pin` is set → full clone + checkout pin.
3. `repo_dir` doesn't exist, `cfg.pin` is empty → shallow clone (`--depth 1`).

**Why pin triggers full clone** (line 61-65): A shallow clone has only the latest commit. If the pin points to an earlier commit, `git checkout pin` fails with "pathspec not in the commit history". A full clone preserves the entire history so any commit can be checked out.

**Why no-pin uses shallow clone** (line 67-69): Speed. Ingesting 10 sources × full clone of each = gigabytes of git history downloaded. Shallow clone downloads only the latest snapshot. When pin is empty (track HEAD), there's no need for history — HEAD is the only commit that matters.

### §4.2 `_clone` — The Internal Method (lines 59-69)

```python
# Learning mode: Understand | Two strategies based on whether a pin is set
def _clone(self, cfg: SourceConfig, repo_dir: Path) -> str:
    if cfg.pin:
        _run(["git", "clone", "--quiet", cfg.url, str(repo_dir)])
        _run(["git", "checkout", cfg.pin], cwd=repo_dir)
        return cfg.pin             # resolved = pin (we checked it out)
    else:
        _run(["git", "clone", "--depth", "1", "--quiet", cfg.url, str(repo_dir)])
        return self._current_commit(repo_dir)   # resolved = HEAD SHA
```

**Return value** is the resolved pin — either the explicitly requested pin (verified by checkout), or the SHA that HEAD resolved to at clone time.

### §4.3 `_current_commit` — HEAD SHA (lines 71-78)

```python
@staticmethod
def _current_commit(repo_dir: Path) -> str:
    result = subprocess.run(
        ["git", "rev-parse", "HEAD"],
        cwd=repo_dir, capture_output=True, text=True, check=True,
    )
    return result.stdout.strip()
```

**`git rev-parse HEAD`** — the canonical way to get the current commit SHA. `check=True` raises `CalledProcessError` if the repo is corrupted (not a git dir, detached HEAD, etc.).

### §4.4 `post_clone_cmd` — The `extra` Hook (lines 37-41)

```python
# Learning mode: MASTER🔵 | Source-specific post-processing without modifying the connector
post_cmd = cfg.extra.get("post_clone_cmd")
if post_cmd and not (dest / ".post_clone_done").exists():
    _run(post_cmd.split(), cwd=repo_dir)
    (dest / ".post_clone_done").touch()
```

**The `.post_clone_done` sentinel file** makes this idempotent: on re-ingest, if the file exists, the post-clone step is skipped. If the manifest verification detects changes, the re-ingest will re-run the full pull (which re-creates `repo_dir`).

**Why this exists**: scabench requires `python checkout_sources.py` to generate the actual `.sol` files from the repo's metadata. Without the hook, the connector would need to know about scabench's internal build process — a leaky abstraction.

> 🔵 **Portable recall — `extra` in action:** `post_clone_cmd` flows through `SourceConfig.extra`, demonstrating the catch-all dict pattern from Session 05. Source-specific hooks bypass the typed interface without breaking it.

### §4.5 `_run` — Subprocess Helper (lines 81-89)

```python
def _run(cmd: list[str], cwd: Path | None = None) -> None:
    result = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True)
    if result.returncode != 0:
        raise ConnectorError(
            f"Command failed: {' '.join(cmd)}\n"
            f"stdout: {result.stdout[-500:]}\n"
            f"stderr: {result.stderr[-500:]}"
        )
```

Truncates stdout/stderr to last 500 chars — enough to see the error, not so much that the exception message is unreadable.

---

## §5 `manual_connector.py` — Staging + Materialization (189 LOC)

### §5.1 `_pull` — The Entry Point (lines 54-100)

```python
# Learning mode: Understand | 2 paths: materialize fresh vs re-use existing
def _pull(self, cfg: SourceConfig, dest: Path) -> PullResult:
    extra = cfg.extra or {}
    staging = extra.get("staging_path")
    if not staging:
        raise ConnectorError(...)

    repo_dir = dest / "repo"
    if repo_dir.exists() or repo_dir.is_symlink():
        pass                                         # idempotent re-run
    else:
        materialize_staging(Path(staging), repo_dir, extra, source_name=cfg.name)

    sol_files = self.find_sol_files(...)
    if not sol_files:
        raise ConnectorError(f"...found 0 .sol files...")

    # Pin: use cfg.pin if set, else fall back to staging dir mtime
    try:
        staging_stat = os.stat(staging)
        pin_marker = datetime.fromtimestamp(staging_stat.st_mtime).strftime("%Y-%m-%d")
    except OSError:
        pin_marker = cfg.pin or "unknown"

    return PullResult(
        source=cfg.name, local_dir=repo_dir,
        resolved_pin=cfg.pin or pin_marker, ...
    )
```

**Key structural differences from git_connector:**
- Uses `cfg.extra` heavily for source-specific config (`staging_path`, `staging_glob`, `materialize`)
- Post-materialization check: raises `ConnectorError` if 0 `.sol` files found (different from git where 0 files is silently OK)
- Pin resolution: no git SHA available → uses `cfg.pin` if set, else falls back to staging directory's `st_mtime`

### §5.2 `materialize_staging` — The 3 Cases (lines 103-158)

```python
# Learning mode: Master | 3 materialization strategies, idempotent re-runs
def materialize_staging(staging: Path, repo_dir: Path, extra: dict, source_name: str) -> None:
    # Resolve glob patterns
    if any(c in str(staging) for c in "*?["):
        matches = [Path(p) for p in glob.glob(str(staging))]
        if len(matches) != 1:
            raise ConnectorError(f"...matched {len(matches)} paths; expected 1...")
        staging = matches[0]

    if not staging.exists():
        raise ConnectorError(f"...staging_path does not exist: {staging}")

    mode = (extra.get("materialize") or "symlink").lower()
    if mode not in ("symlink", "copy"):
        raise ConnectorError(f"...materialize must be 'symlink' or 'copy'...")

    if staging.is_file() and staging.suffix.lower() == ".zip":    # Case 1: zip
        ...
    elif staging.is_dir():                                          # Case 2: directory
        ...
    else:
        raise ConnectorError(f"...neither a directory nor a .zip...")
```

**Three materialization cases:**

| Case | Input | Default mode | What happens |
|------|-------|-------------|-------------|
| 1. Zip | `.zip` file | "symlink" → extract + symlink | Extract to `repo_dir__zip_extracted/`, symlink as `repo_dir/` |
| 2. Directory | Existing dir | "symlink" → link | Symlink the staging dir as `repo_dir/` (or copy) |
| 3. Glob | `*?[` pattern | — | Resolve to single path first, then handle as zip or dir |

**Glob resolution (lines 115-122):** If `staging_path` contains glob characters (`*`, `?`, `[`), resolve via `glob.glob()`. Must match exactly 1 path — 0 matches or 2+ matches both raise errors.

**The `mode` distinction for zips (lines 136-149):**

```python
if mode == "symlink":
    # "symlink" for a zip = extract to a temp dir, symlink as repo_dir
    extract_root = repo_dir.parent / f"{repo_dir.name}__zip_extracted"
    if not extract_root.exists():
        extract_root.mkdir(parents=True)
        _extract_zip(staging, extract_root, source_name=source_name)
    repo_dir.symlink_to(extract_root.resolve())
else:  # copy
    _extract_zip(staging, repo_dir, source_name=source_name)
```

**Why this asymmetry:** Zips can't be "symlinked" — the content must be extracted. The `symlink` mode means "extract to a separate location and symlink as repo_dir", preserving the option to delete the symlink without losing the extracted data.

### §5.3 `_extract_zip` — macOS Cleanup (lines 161-189)

```python
# Learning mode: MASTER🔵 | The DIVE zip had 44,687 files, 22,332 were .sol; rest was macOS noise
def _extract_zip(zip_path: Path, dest: Path, source_name: str) -> None:
    with zipfile.ZipFile(zip_path) as zf:
        for member in zf.namelist():
            if "__MACOSX/" in member or member.endswith(".DS_Store"):
                continue
            target = dest / member
            if not str(target.resolve()).startswith(str(dest.resolve())):
                raise ConnectorError(f"...zip-slip attempt: {member}")   # security
            if member.endswith("/"):
                target.mkdir(parents=True, exist_ok=True)
            else:
                target.parent.mkdir(parents=True, exist_ok=True)
                with zf.open(member) as src, open(target, "wb") as dst:
                    shutil.copyfileobj(src, dst)
```

**`__MACOSX/` stripping** — macOS's `Archive Utility.app` creates `__MACOSX/` resource forks alongside every extracted file. The DIVE zip had ~44,687 total entries, only 22,332 were `.sol` files; the rest were `__MACOSX/` metadata. Without the filter, `find_sol_files` would work correctly (it only looks for `.sol`), but the extracted directory would be cluttered with 22k irrelevant files.

**Zip-slip defense** (lines 178-181): `target.resolve()` resolves symlinks/`..` to an absolute path. If the zip entry has `../../etc/passwd`, `target.resolve()` would point outside `dest`, and the check catches it.

---

## §6 The Full Connector Dispatch Flow (ASCII)

```
config.yaml entry:
  solidifi:
    connector: git
    url: https://github.com/SolidiFI/smart-contracts.git
    include_subdirs: [buggy_contracts]

CLI: sentinel-data ingest --source solidifi

ingest.py:ingest_source("solidifi", cfg, data_dir)
  → source_cfg = SourceConfig(name="solidifi", connector="git", ...)
  → connector = get_connector("git")                     → GitConnector()
  → connector.pull(source_cfg, data_dir/"raw"/"solidifi")

connectors/__init__.py:get_connector("git")
  → _REGISTRY["git"] = GitConnector
  → return GitConnector()

base.py:BaseConnector.pull(cfg, dest)
  → dest.mkdir(parents=True, exist_ok=True)
  → start = time.monotonic()
  → result = self._pull(cfg, dest)                       → GitConnector._pull()

git_connector.py:GitConnector._pull(cfg, dest)
  → cfg.url checked (required)
  → repo_dir = dest / "repo"
  → repo_dir doesn't exist → _clone(cfg, repo_dir)
      → cfg.pin is empty → shallow clone --depth 1
      → return current HEAD SHA via git rev-parse
  → sol_files = find_sol_files(repo_dir, include_subdirs=["buggy_contracts"])
      → only descends into repo_dir/buggy_contracts/
      → returns sorted list of .sol files found
  → PullResult(source="solidifi", resolved_pin="abc123...", sol_files=[...])

base.py:BaseConnector.pull (resumes)
  → result.duration_s = time.monotonic() - start
  → result.fetched_at = "2026-06-14T12:00:00Z"
  → return result

ingest.py:ingest_source (resumes)
  → manifest = IngestionManifest(files=build_file_records(result.sol_files, ...))
  → manifest.save(raw_dir / "ingestion_manifest.json")
  → return manifest
```

vs.

```
config.yaml entry:
  dive:
    connector: manual
    extra:
      staging_path: /net/data/dive/v2.1/solidity_files.zip
      materialize: symlink

ingest.py:ingest_source("dive", cfg, data_dir)
  → source_cfg = SourceConfig(name="dive", connector="manual", extra={...}, ...)
  → connector = get_connector("manual")                  → ManualConnector()
  → connector.pull(source_cfg, data_dir/"raw"/"dive")

base.py:BaseConnector.pull(...)  [same timing wrapper]

manual_connector.py:ManualConnector._pull(cfg, dest)
  → extra = cfg.extra  →  {"staging_path": "/net/data/dive/v2.1/solidity_files.zip", ...}
  → staging = "/net/data/dive/v2.1/solidity_files.zip"
  → is_file() + suffix == .zip → materialize_staging(zip_path, repo_dir, extra, ...)

materialize_staging:
  → staging is a .zip
  → mode = "symlink"
  → extract to repo_dir__zip_extracted/
      → _extract_zip: iterate zf.namelist(), skip __MACOSX/ and .DS_Store
  → symlink repo_dir → repo_dir__zip_extracted/
  → return

ManualConnector._pull (resumes):
  → sol_files = find_sol_files(repo_dir, ...)
  → resolved_pin = cfg.pin or mtime-fallback → "2026-06-10"
  → PullResult(source="dive", resolved_pin="2026-06-10", sol_files=[...])

[back through BaseConnector timing → back to ingest_source → manifest saved]
```

---

## §7 3 Things to Lock In (per global P10-C)

1. **`find_sol_files` is shared by all connectors.** The allowlist (`include_subdirs`) limits the search space; the blocklist (`exclude_subdirs`) filters after the glob. Both are configured per-source in `config.yaml`, not hardcoded in connectors.

2. **Git connector has 2 clone strategies based on pin presence.** Pinned → full clone (need history to checkout the pin). Unpinned → shallow clone (speed). Idempotent re-runs skip cloning entirely.

3. **The manual connector's "symlink" mode for zips is not a symlink.** It extracts to a `__zip_extracted` sibling dir and symlinks that as `repo_dir`. This preserves the option to delete the symlink without re-extracting. The `materialize` option controls behavior (symlink vs copy), not the data format.

---

## §8 Abbreviations introduced this session (per global P12)

- **ABC** — Abstract Base Class (Python's `abc.ABC`; a class that cannot be instantiated directly, only subclassed)
- **Template method** — design pattern where a base class defines the skeleton of an algorithm and subclasses fill in specific steps
- **Factory** — design pattern where a function returns an instance of the correct class based on a string key
- **Sentinel file** — a marker file (`.post_clone_done`) whose presence indicates a one-time operation has completed
- **Zip-slip** — a directory traversal vulnerability where a malicious zip entry uses `../` to write outside the extraction directory

---

## §9 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> In `git_connector._pull()` (line 32-33), if `repo_dir` already exists, the connector skips cloning and reads the current commit via `_current_commit()`. What happens if the upstream repo has new commits since the last ingestion? Is the data silently stale?

### Q2 [Pattern]

> `materialize_staging` has a `mode` parameter ("symlink" or "copy") but zips treat `mode="symlink"` as "extract + symlink" and `mode="copy"` as "extract in place". Why this asymmetry? Why not just extract zips the same way regardless of mode?

### Q3 [Portable🔵]

> In `find_sol_files`, if `include_subdirs=["buggy_contracts", "results"]` and `exclude_subdirs=["results"]`, how are files in `results/` handled? Does include win (the dir is in the search list) or does exclude win (files are filtered after glob)? Is this interaction documented anywhere?

### Q4 [Mechanism]

> The manual connector's `resolved_pin` falls back to `st_mtime` of the staging path when `cfg.pin` is empty (line 87-91). `st_mtime` changes every time the staging dir is touched (e.g., a file is added/deleted). How does this affect the manifest's reproducibility guarantee? If you re-ingest without changing any `.sol` files but `st_mtime` changes, does the manifest differ?

---

## §10 Connections (what to study next)

- **Session 07 — Label folderize:** `connectors/` has a sibling `label_folderize.py` (173 LOC) in the same `ingestion/` package. It creates per-class symlink folders from the ingested `.labels.json` files — the bridge between "raw ingested files" and "framework-ready training data."
- **Session 08 — Preprocessing:** Stage 1b enters after ingestion. The preprocess stage reads the manifest, verifies SHA-256s, and builds the per-source metadata that `_run_preprocess` dispatches to.

---

## §11 What "got it" looks like for Session 06

- [ ] Explain the strategy pattern: how `get_connector("git")` returns a `GitConnector` without `ingest_source` knowing which connector type it is.
- [ ] Contrast the git vs manual connectors: what `_pull()` does differently (clone vs materialize), what stays the same (`find_sol_files`, timing wrapper in `BaseConnector.pull()`).
- [ ] Explain the pin-dependent clone strategy: when is a full clone needed vs a shallow clone, and why.
- [ ] Describe the 3 materialization cases (zip, dir, glob) and explain the `mode` asymmetry for zips.
- [ ] Explain the `extra` dict flow from config.yaml → `_source_config` → `SourceConfig.extra` → connector `_pull()`.
- [ ] Describe how `find_sol_files` applies allowlist vs blocklist and how they interact.

---

**Next:** answer Q1–Q4 in chat. I'll gap-fill (P2), update the **Your answers** sections, and update `session_log.md` for this session. After Q&A, we move to Session 07 (label folderize).
