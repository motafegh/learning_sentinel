# Session 13 — Segmenter + VersionBucketer (`preprocessing/segmenter.py`)

**Date:** 2026-06-14
**Session number:** 13 of 36
**Mode:** Understand (68 lines — version-era classification, contract name extraction, unchecked detection)
**Estimated study time:** 25 min teaching + 10 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **PreprocessingPipeline Step 5** (Session 08): The final step, called AFTER normalization. Segments and classifies but does NOT modify content — the file is already written. Results go into `ContractMeta.version_bucket`, `has_unchecked_block`, `contract_names`.
- **Version buckets** are used by the labeling stage to apply version-specific label rules and by the splitting stage for stratified splits.

---

## §1 The Three Version Buckets (lines 3–9)

```
legacy       < 0.6     Pre-SafeMath era. Integer overflow is implicit (wrapping).
                           No built-in overflow checks.

transitional 0.6–0.7  Solidity 0.6+ added SafeMath-like overflow checks for arithmetic.
                           0.7+ refined the behavior but didn't change the overflow model.

modern       >= 0.8    Solidity 0.8+ made overflow checks the DEFAULT for all arithmetic.
                           Introduced `unchecked { }` block to opt out of overflow checks.
```

**Why these three buckets matter for vulnerability detection:**
- Integer overflow/underflow detection must know whether the target contract compiles with implicit overflow (legacy) or explicit unchecked blocks (modern). A false positive from a legacy rule applied to a modern contract wastes reviewer time.
- The same vulnerability pattern manifests differently across buckets. Legacy `require(a + b > a)` vs. modern `unchecked { a + b }`.

---

## §2 `segment_and_bucket` (lines 35–54)

```python
def segment_and_bucket(source: str, pragma_raw: str) -> Segment:
    names = _CONTRACT_RE.findall(source)
    primary = names[0] if names else "Unknown"
    bucket = _bucket(pragma_raw)
    has_unchecked = bool(_UNCHECKED_RE.search(source))
    return Segment(contract_name=primary, ..., contract_names=names)
```

**Multi-contract file policy:** The full file source is kept as one unit. No per-contract splitting:

> We treat multi-contract files as one unit (the full file) — splitting them into per-contract files would break import chains.

A file with `contract A { ... } contract B is A { ... }` — splitting into two files would break the `is A` inheritance reference. Instead, `contract_names = ["A", "B"]` records all names, and downstream stages can match labels by contract name within a file.

---

## §3 Regex Details

### `_CONTRACT_RE` (line 18)

```python
_CONTRACT_RE  = re.compile(r'\bcontract\s+(\w+)\s*(?:is\s+[^{]+)?\{', re.MULTILINE)
```

Matches:
- `contract Foo {`
- `contract Foo is Bar {`
- `contract Foo is Bar, Baz {`

Does NOT match:
- `abstract contract Foo {` — `abstract ` prefix before `contract`
- `interface Foo {` — keyword mismatch
- `library Foo {` — keyword mismatch

**Why no `abstract` / `interface` / `library`?** The regex was designed for the dominant patterns in the SENTINEL datasets. Adding `abstract\s+contract` would be a one-line fix (docs say Stage 2 will extend this). In practice, most vulnerability-relevant code targets `contract` declarations.

### `_PRAGMA_VER` (line 19)

```python
_PRAGMA_VER   = re.compile(r'(\d+)\.(\d+)\.(\d+)')
```

Extracts the FIRST semver triple from the pragma. For:
- `^0.8.0` → major=0, minor=8 → "modern"
- `>=0.6.0 <0.9.0` → major=0, minor=6 → "transitional"
- `^0.4.0 || ^0.5.0` → major=0, minor=4 → "legacy"
- No match → "legacy" (safe default)

**Note:** `^0.4.0 || ^0.5.0` extracts `0.4.0` and classifies as `legacy`. But 0.5.x is transitional. A contract written for 0.5.x with an `|| ^0.4.0` fallback is misclassified. This is an acknowledged imprecision — the pragma parser extracts the lowest version, and the bucket follows that version.

### `_UNCHECKED_RE` (line 20)

```python
_UNCHECKED_RE = re.compile(r'\bunchecked\s*\{')
```

Detects `unchecked { ... }` blocks. This feature was introduced in Solidity 0.8.0. A `legacy` or `transitional` contract with `has_unchecked_block=True` is unusual but possible (the pragma says <0.8 but someone copy-pasted unchecked code). The segmenter records what it finds without enforcing consistency.

---

## §4 `_bucket()` — The Version Mapper (lines 57–68)

```python
def _bucket(pragma_raw: str) -> str:
    m = _PRAGMA_VER.search(pragma_raw)
    if not m: return "legacy"
    major, minor = int(m.group(1)), int(m.group(2))
    if major == 0 and minor < 6: return "legacy"
    if major == 0 and minor < 8: return "transitional"
    return "modern"
```

Only handles `major == 0` cases (EVM-compatible Solidity is never expected to hit 1.0 in practice). If `major > 0`, falls through to "modern" — which is wrong for theoretical-future Solidity 1.x, but the community has been at 0.x for a decade.

---

## §5 `Segment` Dataclass (lines 23–33)

```python
@dataclass
class Segment:
    contract_name: str          # first contract's name (or "Unknown")
    source: str                 # full file source (unchanged)
    version_bucket: str         # "legacy" | "transitional" | "modern"
    has_unchecked_block: bool   # 0.8+ unchecked {} used?
    pragma_raw: str             # passed through from compiler
    contract_names: list[str]   # ALL contract/interface/library names found
```

---

## §0.6 Exercises (Q1–Q4)

### Q1 — `_bucket` uses `major == 0` checks. What if a future Solidity version is 1.0.0?

All contracts with `major >= 1` fall through to `return "modern"` (line 68). This would misclassify a 1.0.0 contract that behaves like 0.8.x (overflow checks, unchecked blocks) — the bucket would be correct. But a 1.0.0 contract with breaking changes would be treated as "modern" when it might not be. This is speculative — Solidity is still at 0.8.x and the community expects 1.0.0 to be behaviorally similar to 0.8.x.

### Q2 — `_CONTRACT_RE` doesn't match `abstract contract`. A file with `abstract contract Base { }` would NOT have "Base" in `contract_names`. Could this cause downstream issues?

Yes. If a labeling parser tries to match a label to contract "Base" and it's not in `contract_names`, the label is silently dropped. This is an acknowledged gap — the fix is to extend the regex to `(?:abstract\s+)?contract\b`. The severity is low because abstract contracts rarely contain vulnerabilities in isolation (they're base contracts that provide shared functionality).

### Q3 — Multi-contract files: if `A.sol` defines contracts `X`, `Y`, `Z` and a downstream stage finds a bug in `Y`, how does it record the label?

The downstream stage reads `ContractMeta.contract_names = ["X", "Y", "Z"]` and matches labels by contract name within the file's source. The preprocessed file `A.sol` is a single unit — the label is attached to the file + contract name pair, not to the file alone. The splitting stage (Session 17+) must ensure the same file isn't split across train/val/test when only one of its contracts appears in each split.

### Q4 — Why does `Segment` carry `source` (the full file) when the pipeline already writes the file to disk?

The `Segment.source` is never used outside the segmenter. The pipeline reads `segment_and_bucket(content, ...)` but only writes `ContractMeta` fields from the return value. The `source` field is vestigial — it was originally intended for per-contract chunking (split each contract into its own file) but the multi-contract approach was chosen instead. The field remains for back-compat with downstream code that may reference `Segment.source`.
