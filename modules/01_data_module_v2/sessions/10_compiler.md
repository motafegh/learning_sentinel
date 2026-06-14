# Session 10 — Compiler (`preprocessing/compiler.py`)

**Date:** 2026-06-14
**Session number:** 10 of 36
**Mode:** Deep dive (191 lines — two-pass solc compile, pragma resolution, version solver)
**Estimated study time:** 45 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **PreprocessingPipeline Step 2** (Session 08): `compile_contract(compile_target)` is called with the flattened `.sol` file path. If it fails, the file is dropped to `dropped.csv` with `reason: "compile_failed"`.
- **Temp file protocol** (Session 08): the flattener writes `.sentinel_compile_` temp files in the source directory so relative imports resolve correctly.
- **solc-select**: the project uses `solc-select` to manage multiple Solidity compiler versions. Binaries live at `~/.solc-select/artifacts/solc-<version>/solc-<version>`.

---

## §1 `CompileResult` — The Output Dataclass (lines 28–40)

```python
@dataclass
class CompileResult:
    success: bool
    solc_version: str          # version actually used (e.g. "0.8.17")
    pragma_raw: str            # pragma string with whitespace stripped (e.g. "^0.8.0")
    error: str = ""            # last error message (truncated to 500 chars, then 300 in PipelineResult)
    attempted_versions: list[str] = None  # all versions tried in order
```

**Key** — the design is **no-raise**: every failure path populates `success=False` with the error in `.error`. The pipeline never catches exceptions from the compiler. This keeps the pipeline robust: a malformed `.sol` file that crashes solc is just a `CompileResult(success=False, error="...")`.

**`attempted_versions`** traces the compiler's search path. In `dropped.csv`, this becomes a comma-separated string under `attempted_solc_versions`. When debugging a compile failure, this list tells you exactly which versions were tried and in what order.

---

## §2 `compile_contract()` — The Two-Pass Strategy (lines 43–101)

```
compile_contract(sol_path)
         │
         ├── 1. Read source + extract pragma
         │
         ├── NO PRAGMA?
         │   └── return CompileResult(success=False, error="no pragma solidity found")
         │
         ├── PASS 1: Try EXACT version
         │   ├── _parse_version(pragma_raw)  → "0.8.17" or ""
         │   ├── If version found AND binary installed:
         │   │     _run_solc(bin_path, sol_path)
         │   │     ├── success  → return CompileResult(success=True, version=requested)
         │   │     └── fail     → save last_err, fall through to Pass 2
         │   └── If no binary for exact version: fall through
         │
         ├── PASS 2: Try NEAREST SATISFYING versions
         │   ├── _available_versions()  → list of all installed solc versions
         │   ├── _satisfying_versions(pragma, available)  → filtered + sorted newest-first
         │   ├── For each candidate (skip if already tried in Pass 1):
         │   │     _run_solc(bin_path, sol_path)
         │   │     ├── success  → return CompileResult(success=True, version=candidate)
         │   │     └── fail     → save last_err, continue
         │   └── All candidates exhausted
         │
         └── return CompileResult(success=False,
                                   error=f"all versions failed; last error: {last_err[:300]}",
                                   attempted_versions=attempted)
```

### Pass 1 — Exact Version (lines 65–77)

`_parse_version(pragma_raw)` extracts a single clean version from the pragma:

| Pragma | `_parse_version` result | Pass 1 behavior |
|---|---|---|
| `0.4.25` | `"0.4.25"` | Try exact 0.4.25 |
| `=0.4.25` | `"0.4.25"` | Try exact 0.4.25 |
| `^0.8.0` | `"0.8.0"` | Try exact 0.8.0 |
| `~0.6.12` | `"0.6.12"` | Try exact 0.6.12 |
| `>=0.4.0 <0.9.0` | `""` (ambiguous range) | Skip Pass 1 entirely |
| `^0.4.0 \|\| ^0.5.0` | `""` (OR pattern) | Skip Pass 1 entirely |

**Bug fix from Phase 5** (lines 7–9): The original retry script compiled `0.4.25` (exact pragma) with the wrong solc version. Pass 1 ensures the EXACT version is tried first, with no version relaxation. This is critical: a contract written for `0.4.25` may use features that break in `0.4.26`.

### Pass 2 — Nearest Satisfying (lines 79–93)

If Pass 1 fails (version not installed, or compilation error), Pass 2 relaxes the constraint:

1. `_available_versions()`: reads `~/.solc-select/artifacts/solc-*` directories
2. `_satisfying_versions(pragma, available)`: filters + sorts **newest first**
3. Tries each candidate, skipping any already tried in Pass 1

**Why newest first?** Two reasons:
- Newer solc versions have better error messages (helpful for debugging)
- In practice, code written for older Solidity almost always compiles on a newer version (backward compatibility). Trying the latest satisfying version first means you hit a working compiler faster.
- The `--bin` output (EVM bytecode) is what matters downstream — small version differences rarely change the bytecode for the same source.

---

## §3 Helper Functions

### `_extract_pragma` (lines 106–113)

```python
def _extract_pragma(source: str) -> str:
    m = _PRAGMA_RE.search(source)
    if not m: return ""
    return re.sub(r"\s+", "", m.group(1))
```

**Bug fix: spaced pragmas** (line 8, lines 112–113). Solidity allows whitespace between tokens in pragma statements, like `pragma solidity ^ 0.4 . 9 ;`. The regex captures `"^ 0.4 . 9"`, which is unparseable. Stripping all whitespace reduces it to `"^0.4.9"`.

**What it handles:**
- `pragma solidity ^0.8.0;` → `^0.8.0`
- `pragma solidity >=0.4.0 <0.9.0;` → `>=0.4.0<0.9.0`
- `pragma solidity ^ 0.8 . 0 ;` → `^0.8.0`
- Multiple pragma lines (`.search` finds first match)

**What it does NOT handle:**
- No pragma → returns `""` → compile fails with "no pragma solidity found"
- Multiple conflicting pragmas (e.g., `^0.4.0` in one line and `^0.5.0` later) — only the first match is used

### `_parse_version` (lines 116–126)

```python
def _parse_version(pragma: str) -> str:
    # exact version: `0.4.25` or `=0.4.25`
    m = re.fullmatch(r"=?(\d+\.\d+\.\d+)", pragma)
    if m: return m.group(1)
    # caret / tilde: `^0.8.0`, `~0.6.12`
    m = re.fullmatch(r"[\^~>=]*(\d+\.\d+\.\d+)", pragma)
    if m: return m.group(1)
    return ""
```

**Design:** Only returns a non-empty string for simple pragmas that resolve to a single version. For ranges (`>=0.4.0 <0.9.0`) or OR patterns (`^0.4.0 || ^0.5.0`), returns `""` and delegates to `_satisfying_versions` in Pass 2.

**Second regex** (`[\^~>=]*(\d+\.\d+\.\d+)`) accepts `^0.8.0`, `~0.6.12`, `>=0.5.0`. The `*` quantifier means it also matches `>>>0.8.0` (degenerate but harmless — the capture group still extracts the version).

### `_satisfying_versions` (lines 129–150)

```
_satisfying_versions("^0.8.0", ["0.4.24", "0.5.0", "0.8.0", "0.8.1", "0.8.17"])
    → floor = (0, 8, 0), ceiling = None
    → satisfied: 0.8.0, 0.8.1, 0.8.17
    → reversed (newest first): ["0.8.17", "0.8.1", "0.8.0"]
```

```
_satisfying_versions(">=0.4.0<0.9.0", ["0.4.0", "0.5.0", "0.8.17", "0.9.0"])
    → floor = (0, 4, 0), ceiling = (0, 9, 0)
    → satisfied: 0.4.0, 0.5.0, 0.8.17
    → reversed: ["0.8.17", "0.5.0", "0.4.0"]
```

**Ceiling extraction:** The regex `<(\d+\.\d+\.\d+)` catches the upper bound from double-bound pragmas like `>=0.4.0 <0.9.0`. Single-bound pragmas like `^0.8.0` have no ceiling.

### `_available_versions` (lines 153–163)

```python
def _available_versions() -> list[str]:
    if not _SOLC_ARTIFACTS.exists(): return []
    vers = []
    for d in _SOLC_ARTIFACTS.iterdir():
        if d.is_dir() and d.name.startswith("solc-"):
            ver = d.name[len("solc-"):]
            if re.fullmatch(r"\d+\.\d+\.\d+", ver):
                vers.append(ver)
    return sorted(vers, key=lambda v: tuple(int(x) for x in v.split(".")))
```

**Directory structure:** `~/.solc-select/artifacts/solc-0.8.17/solc-0.8.17`

**Sorted ascending** (oldest first). In `_satisfying_versions`, this list is `reversed()` for newest-first candidate ordering.

**Edge case:** If `_SOLC_ARTIFACTS` doesn't exist (solc-select not installed), returns `[]` → Pass 2 has zero candidates → every compile fails. The pipeline crippled until solc versions are installed.

### `_solc_binary` (lines 166–170)

```python
def _solc_binary(version: str) -> Path | None:
    p = _SOLC_ARTIFACTS / f"solc-{version}" / f"solc-{version}"
    return p if p.exists() else None
```

Simple existence check. Returns `None` if the version directory exists but the binary doesn't (partial installation, corruption).

### `_run_solc` (lines 173–191)

```python
def _run_solc(bin_path: Path, sol_path: Path) -> tuple[bool, str]:
    allow_root = sol_path.parent.parent
    result = subprocess.run(
        [str(bin_path), "--bin", str(sol_path),
         "--allow-paths", str(allow_root)],
        capture_output=True, text=True, timeout=30,
    )
    if result.returncode == 0: return True, ""
    return False, (result.stderr or result.stdout)[-500:]
```

**Key flags:**
- `--bin`: Compile to EVM bytecode (no ABI, no metadata — just the runtime bytecode)
- `--allow-paths sol_path.parent.parent`: Allows relative imports to resolve up to two directory levels. Without this, solc 0.8.x blocks imports crossing outside the file's directory.

**Error handling:**
- 30-second timeout (prevents infinite loops on pathological files)
- On failure, takes LAST 500 chars of stderr (fallback to stdout if stderr empty)
- The pipeline then truncates further to 300 chars in the dropped.csv row

---

## §4 Compiler in the Pipeline

```
PreprocessingPipeline._process_one:
    Step 2:
        compile_target = sol_path  (or temp file if flattened content differs)
        compile_res = compile_contract(compile_target)

        if not compile_res.success:
            return None, None, {
                "source": self.source_name,
                "original_path": str(rel),
                "pragma": compile_res.pragma_raw,
                "reason": "compile_failed",
                "error": compile_res.error[:300],
                "attempted_solc_versions": ",".join(compile_res.attempted_versions),
            }
```

**The compile step is the bottleneck.** The entire pipeline is serial (single-process), and `subprocess.run(solc, timeout=30)` is the only I/O-bound step per file. For a source with 22,000 files, compile dominates the runtime.

**Why no caching?** Compilation is deterministic for the same source+version. A SHA-256-based compile cache (`_cache[sol_sha + solc_version] = result`) would eliminate redundant compilations when the same file appears in multiple class folders or when retry-failed is called with the same version set. Deferred to v2.x.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — `_parse_version("^0.4.0 || ^0.5.0")` returns `""`. Trace Pass 2.

The pragma `^0.4.0 || ^0.5.0` means "satisfied by 0.4.x OR 0.5.x". `_parse_version` returns `""` (the `||` pattern doesn't match any regex). Pass 1 is skipped entirely.

Pass 2 calls `_satisfying_versions("^0.4.0||^0.5.0", available)`. The floor regex extracts `"0.4.0"` (first match from the `||` expression). No ceiling. So `satisfies("0.8.17")` → `(0,8,17) >= (0,4,0)` → True. Solc 0.8.17 is tried first (newest-first). It will likely fail for most `^0.4.0` code. The next newest version that satisfies the floor is tried next, and so on.

This is **suboptimal** because `^0.8.17` and `^0.5.0` are both tried before `^0.4.0` versions, even though the pragma explicitly allows 0.4.x. The compiler will waste many iterations trying incompatible versions before hitting a working one. A better approach would parse the OR pattern into separate ranges and search each independently.

### Q2 — Why `--bin` instead of `--combined-json` or `--abi`?

`--bin` produces the raw EVM runtime bytecode. The compile step's only goal is to verify that the file compiles (success/failure). The bytecode itself is NOT used by any downstream stage — dedup (Step 3) uses the flattened source text, and normalize+segment (Steps 4-5) use the source. So why compile at all?

Compilation is a **quality gate**: files that don't compile are dropped. The rationale: if a contract doesn't compile, its source may be incomplete, malformed, or have unresolvable imports. Training a vulnerability detector on uncompilable code would introduce noise. The `--bin` flag is the simplest, fastest compile target (`--abi` or `--combined-json` would do extra work that's thrown away).

### Q3 — The `--allow-paths` argument uses `sol_path.parent.parent`. What breaks if a file is at the repo root?

If `sol_path = data/raw/dive/repo/X.sol`:
- `sol_path.parent` = `data/raw/dive/repo/`
- `sol_path.parent.parent` = `data/raw/dive/`

If `sol_path = data/raw/dive/repo/deep/nested/X.sol`:
- `parent.parent` = `data/raw/dive/repo/`

If `sol_path = data/raw/dive/X.sol` (flat file at `raw_dir` root, no repo/ subdir):
- `parent.parent` = `data/raw/` — which is the parent of `raw_dir`, giving access to OTHER sources' raw directories. This is a security concern (cross-source import resolution) but unlikely to cause bugs in practice since solc only resolves imports that are actually referenced.

The assumption is that files always live at least 2 levels deep in the directory hierarchy (`raw/<source>/repo/` or `raw/<source>/__source__/`).

### Q4 — `_available_versions` reads from `~/.solc-select/artifacts/`. What's the bootstrap process?

If `~/.solc-select/artifacts/` doesn't exist, `_available_versions()` returns `[]` → Pass 2 tries no versions → every compile fails with "all versions failed". The pipeline is completely non-functional.

The bootstrap process (outside this module):
1. Install solc-select: `pip install solc-select`
2. Install needed versions: `solc-select install 0.4.0 0.4.25 0.5.0 0.6.12 0.7.6 0.8.0 0.8.17 0.8.26`
3. The Stage 1 setup script (`scripts/install_solc_versions.sh`) runs these commands
4. After installation, `_SOLC_ARTIFACTS` contains the version directories and the compiler module works

The design manualizes solc version management — the data module doesn't auto-install versions. This is intentional: version installation is a user/sysadmin responsibility, not a library concern.
