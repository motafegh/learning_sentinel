# Session 11 — Flattener (`preprocessing/flattener.py`)

**Date:** 2026-06-14
**Session number:** 11 of 36
**Mode:** Deep dive (311 lines — import resolution, two-stage fallback, transitive strip)
**Estimated study time:** 55 min teaching + 20 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **PreprocessingPipeline Step 1** (Session 08): `flatten_contract(sol_path)` is called first. Its output `FlattenResult.content` feeds Steps 2–5. If flattening modified content, the compile step writes a temp file (`pipeline.py:130–145`).
- **Temp file cleanup** (Session 08): `pipeline.py:156–163` globs for `.sentinel_stripped.sol` files after each file — those are the transitive strip files the flattener may have written.
- **Compiler** (Session 10): `_pick_solc(pragma)` uses the same version resolution as the compiler — `_parse_version` then `_satisfying_versions`.

---

## §1 The Problem — Why Two Fallbacks?

The docstring tells the story (lines 9–14):

> DeFiHackLabs: 717/738 PoCs import `forge-std/Test.sol` which is not present in the cloned repo (forge submodule not pulled).

Three paths a `.sol` file can take:

```
                     ┌───────────────────────┐
                     │     flatten_contract   │
                     │       (sol_path)       │
                     └───────────┬───────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Has import statements?  │
                    │  _IMPORT_RE.search()     │
                    └──────┬─────────────┬─────┘
                       NO  │             │  YES
                           │             │
               ┌───────────▼┐   ┌────────▼──────────┐
               │  skipped_  │   │   _pick_solc()    │
               │  no_imports │   │  Try solc --flatten│
               └────────────┘   └────────┬──────────┘
                                         │
                            ┌────────────▼────────────┐
                            │  solc --flatten OK?     │
                            └──────┬─────────────┬────┘
                              YES  │             │  NO
                                   │             │
                    ┌──────────────▼┐   ┌────────▼──────────┐
                    │   "flattened" │   │   Strip fallback  │
                    └───────────────┘   │  _strip_unresolved│
                                        │  _imports_recursive│
                                        └────────┬──────────┘
                                                 │
                                   ┌─────────────▼─────────────┐
                                   │  Stripped any imports?    │
                                   │  AND all were unresolved? │
                                   └──────┬──────────────┬─────┘
                                     YES  │              │  NO
                                          │              │
                              ┌───────────▼┐     ┌───────▼──────┐
                              │ "stripped_ │     │ "skipped_   │
                              │ unresolved_│     │ error" —    │
                              │ imports"   │     │ use original│
                              │ + inherit  │     │ source      │
                              │ strip      │     │             │
                              └────────────┘     └─────────────┘
```

### Path A — No imports (63% of DeFiHackLabs)
`skipped_no_imports`. Content = original source. Fast path — no I/O beyond reading the file.

### Path B — solc --flatten succeeds
`flattened`. Content = solc's stdout. This is the clean path. The flattener uses `_pick_solc(pragma)` to select the matching solc version, then runs `solc --flatten file.sol`.

### Path C — solc --flatten fails, strip fallback
`stripped_unresolved_imports`. Content = source with unresolvable import lines removed, plus inheritance parents stripped, plus transitive sub-strips applied. This is the "controlled lossy" path that saves DeFiHackLabs from 717 dropped files.

### Path D — Both fail
`skipped_error`. Content = original source. The compile step will likely fail too, but the pipeline lets the compiler make the final decision.

---

## §2 `FlattenResult` Dataclass (lines 26–31)

```python
@dataclass
class FlattenResult:
    content: str
    flatten_status: str   # "flattened" | "skipped_no_imports" | "skipped_error"
                          # | "stripped_unresolved_imports"
    error: str = ""
```

Four possible `flatten_status` values. The pipeline uses this in `pipeline.py:132`:
```python
if flat.content != source and flat.flatten_status != "skipped_no_imports":
```
Meaning: if the flattener changed the content AND it's not the "no imports" no-op, we need to write a temp file for compilation.

---

## §3 The Strip Fallback — `_strip_unresolved_imports_recursive` (lines 145–236)

### Resolution Rules (lines 153–157)

```
Import form            → Resolution
────────────────────────────────────────────────────
import "x.sol"         → sol_path.parent / "x.sol"
import "dir/x.sol"     → sol_path.parent / "dir/x.sol"
import "../foo/x.sol"  → sol_path.parent / "../foo/x.sol"
import "forge-std/x.sol" → sol_path.parent / "forge-std/x.sol"
```

If the target file exists on disk → the import is kept AND recursively stripped.
If the target file does NOT exist → the import line is removed entirely.

### Recursive Processing (lines 193–211)

When an import target IS a relative path that resolves on disk (`target.startswith(".") or candidate.exists()`):

```python
sub_stripped, sub_n, sub_unresolved, sub_symbols, _ = _strip_unresolved_imports_recursive(
    sub_source, candidate_resolved, seen, sub_strips
)
```

This handles the DeFiHackLabs pattern: `X.sol` imports `"./interface.sol"` (exists) which imports `"forge-std/Test.sol"` (doesn't exist). The recursive call strips the forge-std import from `interface.sol`, and the modified `interface.sol` content is stored in `sub_strips[candidate_resolved]`.

### Transitive Strip Writing (lines 111–116)

When the recursive strip modified shared files (`sub_strips` non-empty), `apply_sub_strips_to_source` writes those modified contents to temp files alongside the originals:

```python
from ._transitive_strip import apply_sub_strips_to_source
stripped, siblings = apply_sub_strips_to_source(
    stripped, sol_path, sub_strips
)
```

The temp files are named `.sentinel_stripped.sol` (the pattern cleaned up by `pipeline.py:156–163`).

### Symbol Collection (lines 217–231)

When stripping an import, the flattener collects which symbols that import brought in:

```python
# `import {Test, console} from "forge-std/Test.sol";` → syms_str = "{Test, console}"
# `import * as F from "forge-std/Test.sol";`           → syms_str = "* as F"
# `import "forge-std/Test.sol";`                        → syms_str = None  (bare import)
```

For bare imports (610/738 DeFiHackLabs PoCs), the flattener uses `_ASSUMED_BARE_IMPORT_SYMBOLS`: `{"Test", "console", "console2", "Vm", "ScriptUtils"}`.

---

## §4 `_extract_imported_symbols` (lines 239–265)

Three import syntax forms:

```python
# Named imports: {A, B as C, D}  → ['A', 'C', 'D']
# Namespace import: * as F       → ['F']
# Legacy single import: A         → ['A']
```

The `as` aliasing keeps the final name (`A as B` → `B`), because the alias is what appears in inheritance lists.

---

## §5 `_strip_unresolved_inheritance` (lines 268–292)

After stripping imports that brought in symbols like `Test`, contracts that inherited `Test` now have a dangling reference:

```solidity
// Before: contract MyTest is Test, Base {
// After strip:  (Test removed from imports, but still in inheritance list)
// After inheritance strip: contract MyTest is Base {
```

The regex `_CONTRACT_INHERIT_RE` matches `contract Foo is A, B, C {`. For each matched inheritance list, parents present in `removed_symbols` are removed. If no parents remain, the `is` clause is dropped entirely.

**Edge case:** Identically-named symbols from different sources. If `Test` is both a forge-std symbol and a locally-defined contract, the inheritance strip would incorrectly remove the local reference. The comment at lines 228–231 acknowledges this: "being too aggressive here just costs us a few extra dropped files, not silent corruption."

---

## §6 `_pick_solc` (lines 295–311)

```python
def _pick_solc(pragma: str):
    available = _available_versions()
    requested = _parse_version(pragma)
    if requested:
        b = _solc_binary(requested)
        if b: return b
    candidates = _satisfying_versions(pragma, available)
    for ver in reversed(candidates):  # OLDEST first (reversed from compiler's newest-first)
        ...
```

**Note the version order difference from the compiler:** The compiler tries newest-first in Pass 2 (`candidates` reversed). The flattener tries OLDEST first (`reversed(candidates)` where `candidates` is already newest-first). This is intentional — `solc --flatten` tends to work better with older versions that are more permissive about import resolution.

---

## §7 Pipeline Context

```
pipeline.py:_process_one:
    flat = flatten_contract(sol_path)                         # Step 1

    if flat.content != source and flat.status != "skipped_no_imports":
        tmp = sol_path.parent / f".sentinel_compile_{stem}_{mtime}.sol"
        tmp.write_text(flat.content)                          # Write temp for compile
        compile_target = tmp
        # ... cleanup in try/finally ...

    compile_res = compile_contract(compile_target)             # Step 2
    # ... cleanup .sentinel_stripped.sol siblings ...
```

**Critical interaction:** The flattener may write `.sentinel_stripped.sol` files (via `_transitive_strip`). These must be cleaned up before the next file. The cleanup globs `sol_path.parent.glob("*.sentinel_stripped.sol")` — this uses the temp file's parent directory, which is the original source directory, ensuring we only clean up files we created.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — solc --flatten with `_pick_solc` tries oldest-first. Why oldest-first for flatten but newest-first for compile?

The flattener tries the OLDEST satisfying version first. The compiler (Pass 2) tries NEWEST first.

**Reason:** `solc --flatten` has a different success profile than `solc --bin`. Older solc versions are more permissive about import resolution — they don't enforce the "file outside allowed directories" check as strictly as 0.8.x. A file written for 0.6.x that imports from a parent directory will likely flatten with 0.6.x but might fail with 0.8.x's stricter path checks. The flattener prioritizes "any version that produces output" over "the version closest to the file's pragma."

By contrast, `solc --bin` (compilation) tries the closest version first because newer versions give better error messages and the bytecode output matters less than the compile-success signal.

### Q2 — The `_CONTRACT_INHERIT_RE` regex matches `contract Foo is A, B, C {`. What about `abstract contract Foo is A {` or `interface Bar is A {`?

The regex `r"(\bcontract\s+\w+\s+is\s+)([^{]+)( \{)"` only matches `contract`, not `abstract contract`, `interface`, or `library`. If an interface or abstract contract inherits from a stripped import's symbol, the inheritance parent is NOT removed, and the compile step fails with "Identifier not found."

This is an acknowledged gap. It doesn't appear in practice because forge-std imports primarily bring in `Test` (a contract) and `console` (a library used with `Hardhat.sol`/`Vm.sol` is an interface). The DeFiHackLabs PoCs overwhelmingly use `contract X is Test {`, which matches the regex.

### Q3 — The `sub_strips` dict tracks paths that were recursively stripped. What happens if the same shared file is encountered by two different top-level `.sol` files?

If `X.sol` imports `shared.sol` (which imports forge-std) and `Y.sol` also imports `shared.sol`, the first processing of `shared.sol` stores its stripped content in `sub_strips[candidate_resolved]`. The second encounter hits `candidate_resolved in sub_strips` at line 201 and returns early without re-processing. This means `shared.sol`'s stripped content is only written once, and both X.sol and Y.sol's rewritten imports point to the same `.sentinel_stripped.sol` file. If the pipeline processes Y.sol on a different run (different `PreprocessingPipeline` instance), the temp file from X.sol's run is already cleaned up, and Y.sol re-processes `shared.sol` fresh.

### Q4 — `_ASSUMED_BARE_IMPORT_SYMBOLS` is a frozenset of forge-std-specific symbols. What about Hardhat, OpenZeppelin, or other bare imports?

A `import "@openzeppelin/contracts/token/ERC20.sol";` (bare import, no named symbols) would be stripped, and the only assumed symbols are forge-std ones. The inheritance strip would NOT remove OpenZeppelin's `ERC20` from the inheritance list. The compile step would fail with "Identifier not found."

This is a forge-std-centric design. The justification: DeFiHackLabs is the primary source that uses bare imports, and 656/738 DeFiHackLabs PoCs use forge-std. Other sources (SolidiFI, smartbugs_curated) use fully-qualified imports or no imports. If a new source with hardhat bare imports is added, `_ASSUMED_BARE_IMPORT_SYMBOLS` needs updating.
