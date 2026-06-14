# Session 09 — Deduplicator (`preprocessing/deduplicator.py`)

**Date:** 2026-06-14
**Session number:** 09 of 36
**Mode:** Deep dive (standalone 114-line module — 3-level stateful dedup)
**Estimated study time:** 40 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **PreprocessingPipeline** (Session 08): Step 3 calls `Deduplicator.process()` after flatten+compile and before normalize+segment
- **Pipeline statefulness**: the `_dedup` member is created once in `PreprocessingPipeline.__init__` and persists across all files in a `run()` call
- **SHA-256 as identity**: output files are named `<sha256>.sol`; dedup is the reason SHA-256 becomes the canonical filename

---

## §1 `DedupRecord` — The Outcome Dataclass (lines 26–33)

```python
@dataclass
class DedupRecord:
    sha256: str
    dedup_group_id: str
    is_duplicate: bool
    duplicate_of: str
```

| Field | Value when unique | Value when duplicate |
|---|---|---|
| `sha256` | SHA-256 of this file's content | Same |
| `dedup_group_id` | Same as `sha256` | SHA-256 of the canonical (first-seen) file |
| `is_duplicate` | `False` | `True` |
| `duplicate_of` | `""` | SHA-256 of the canonical file |

**Key:** `dedup_group_id` is the "cluster key." All files that are duplicates of each other share the same `dedup_group_id`. The canonical (first-seen) file IS the cluster representative — its `dedup_group_id` = its own `sha256`.

---

## §2 `_normalize_for_dedup` — Level 3's Secret Sauce (lines 36–45)

```python
def _normalize_for_dedup(content: str) -> str:
    out = _BLOCK_CMT.sub('', content)      # strip /* ... */
    out = _LINE_CMT.sub('', out)            # strip // ...
    out = _WHITESPACE.sub(' ', out).strip() # collapse whitespace
    return out
```

Three regexes:

| Regex | Pattern | What it strips |
|---|---|---|
| `_BLOCK_CMT` | `/\*.*?\*/` with `re.DOTALL` | Multi-line comments `/* ... */` |
| `_LINE_CMT` | `//[^\n]*` | Single-line comments `// ...` |
| `_WHITESPACE` | `\s+` → `' '` | All whitespace collapsed to single space |

**What is NOT stripped:**
- String literals (comments inside strings are preserved)
- Identifiers (function names, variable names)
- Ethereum addresses (0x...)

This catches **copy-paste-with-comment-edits**: two contracts where someone changed the NatSpec comments but the code is identical.

---

## §3 `Deduplicator` — The 3 Levels (lines 48–108)

### State (lines 51–57)

```python
self._seen_sha: dict[str, Path]   = {}  # sha256 → canonical path
self._seen_addr: dict[str, str]   = {}  # ethereum address → first sha having it
self._seen_norm: dict[str, str]   = {}  # normalized-hash → first sha with that norm
```

Three separate dicts, one per level. ALL grow monotonically across the pipeline run.

### `process(content, path)` — The 3 Barriers (lines 59–108)

```
                    process(content, path)
                            │
                    sha = sha256(content)
                            │
                ┌───────────▼───────────┐
                │   Level 1: EXACT      │
                │   sha in _seen_sha?   │
                └───────┬──────┬────────┘
                   YES  │      │  NO
                        │      │
                  return │   _seen_sha[sha] = path
                  DUP    │      │
                    ┌────▼──────▼────────┐
                    │  Level 2: ADDRESS  │
                    │  extract addresses │
                    │  any in _seen_addr?│
                    └───────┬──────┬─────┘
                       YES  │      │  NO
                            │      │
                     return │   register all
                     DUP    │   addrs → _seen_addr
                            │      │
                     ┌──────▼──────▼─────┐
                     │  Level 3: NORMAL  │
                     │  strip comments   │
                     │  collapse ws      │
                     │  norm_hash in     │
                     │  _seen_norm?      │
                     └───────┬──────┬────┘
                        YES  │      │  NO
                             │      │
                      return │   _seen_norm[norm_hash] = sha
                      DUP    │      │
                        ┌────▼──────▼──┐
                        │   UNIQUE     │
                        │   return     │
                        │   DedupRecord│
                        │  (is_dup=False) │
                        └──────────────┘
```

### Level 1 — Exact SHA-256 (lines 67–75)

The fastest check: byte-for-byte identical content. If the SHA-256 matches a previously-seen file, it's an exact duplicate.

**Why this is safe:** The pipeline feeds flattened content (after import resolution) into the deduplicator. Two contracts that were different in raw form (different directory structure, different import paths) will produce the same flattened content if the code body is identical. SHA-256 collision is computationally infeasible.

**Edge case:** The same `.sol` file appearing in multiple class folders (DIVE's symlink structure). Flattened content is identical → Level 1 catches it on the second encounter.

### Level 2 — Address-Level (lines 78–89)

Extracts all Ethereum addresses (`0x` + 40 hex characters) from the content. If ANY address was seen in a previous file, the current file is a duplicate.

**Rationale:** Many smart contracts share the same core logic but differ in comments, variable names, or minor formatting. If they reference the same Ethereum address (e.g., a known router, a token contract), they're likely the same contract with superficial edits.

**Conservative strategy:** "If any address matches, it's a duplicate." This WILL produce false positives — two different contracts that happen to reference the same popular address (e.g., `0x0000000000000000000000000000000000000000` or a Uniswap router). This is intentional: **better to drop 10 false duplicates than to keep 1 near-duplicate that pollutes the training set.**

**Address registration (line 88–89):**
```python
for addr in addrs:
    self._seen_addr[addr] = sha
```
ALL addresses in the file get registered, not just the first one. This means a later file with ANY of these addresses will be caught.

### Level 3 — Normalized-Text Hash (lines 92–101)

Strips comments and collapses whitespace via `_normalize_for_dedup()`, then hashes the normalized text. If the normalized hash matches a previously-seen file, it's a duplicate.

**Catches:** Copy-paste-with-comment-edits. Someone takes contract A, adds/removes NatSpec comments, maybe reformats whitespace, calls it contract B. Level 1 misses (different sha, content differs). Level 2 might miss (no addresses, or different addresses). Level 3 catches it because after stripping comments+whitespace, the code is identical.

**What Level 3 does NOT catch:**
- Identifier renaming (e.g., `withdraw` → `payOut`)
- Logic changes (different function bodies)
- Added/removed functions
- For these, you'd need semantic dedup (AST comparison, embedding similarity) — deferred to v2.x

---

## §4 Why Order Matters

The three levels form a **cost-increasing cascade**:

| Level | Cost | Catch rate |
|---|---|---|
| 1 — SHA-256 | O(n) hash, O(1) dict lookup | Exact duplicates |
| 2 — Address regex | O(n) regex scan of content | Address-sharing near-dups |
| 3 — Normalize | O(n) regex scan + O(n) hash | Comment-only near-dups |

**Early termination:** If Level 1 catches it, Levels 2 and 3 never run. For most duplicates (especially in DIVE's symlink structure), Level 1 suffices. Level 2 and 3 only run for files that pass Level 1 — a relatively small subset.

**Group ID assignment:** The `dedup_group_id` is always the sha of the FIRST file that triggered any level. If Level 2 triggers, the group ID comes from `_seen_addr[addr]`. If Level 3 triggers, it comes from `_seen_norm[norm_hash]`. If Level 1 triggers, `duplicate_of = sha` (self — the exact same SHA means the canonical IS the same content hash).

---

## §5 The "No Identifier Lowercasing" Tradeoff

The docstring is explicit (lines 8–10):

> Identifier lowercasing is intentionally NOT applied — it would collapse semantically distinct function names (e.g. `reentrantWithdraw ≠ reentrancyWithdraw`).

This is a **conscious tradeoff**: higher precision (fewer false duplicates) at the cost of lower recall (more near-duplicates slip through).

**What would lowercasing catch?** Two contracts where a developer renamed `withdraw` → `Withdraw` or `SAFE_MINT` → `safe_mint`. These would have different normalized hashes but the same lowercased hash.

**Why not do it?** In the vulnerability detection context, function names carry semantic signal. Two contracts where one has a typo in a function name (`reentrantWithdraw` vs `reentrancyWithdraw`) are NOT the same vulnerability — the typo itself might be the bug. Collapsing them would create label noise.

**Alternative considered:** Levenshtein-based near-dedup (edit distance on normalized AST). Deferred because the 38.8% duplication in BCCC was mostly exact or comment-only, which Levels 1-3 handle.

---

## §6 Dedup in the Pipeline Context

```
PreprocessingPipeline._process_one(sol_path, raw_base):
    ...
    flat = flatten_contract(sol_path)              # Step 1
    compile_res = compile_contract(compile_target)  # Step 2

    dedup_rec = self._dedup.process(flat.content, sol_path)  # Step 3
    if dedup_rec.is_duplicate:
        return None, None, {
            "source": self.source_name,
            "reason": "duplicate",
            "duplicate_of": dedup_rec.duplicate_of,
            ...
        }
```

**Critical detail:** The `Deduplicator` receives **flattened content** (`flat.content`), not raw source. This is intentional:
- Two contracts from different repos that are identical after import resolution get caught by Level 1
- Contracts that differ only in import paths but have identical flattened bodies get caught
- The SHA-256 of the flattened content becomes the output filename

**Why not dedup on raw content?** Let's say `A.sol` imports `X.sol` and `B.sol` has X's content inlined. Raw SHA-256 differs. Flattened SHA-256 is identical → caught. This massively reduces the footprint because the same dependency contract (OpenZeppelin's `Ownable.sol`, for instance) appears in hundreds of repos. Without flatten-level dedup, `Ownable.sol` would appear thousands of times in the preprocessed output. With it, it appears once per unique version.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — `_seen_sha` maps to `Path`, but where is it used?

```python
self._seen_sha: dict[str, Path] = {}
```

The `Path` value (the first file's path) is stored but never read in the pipeline. The pipeline only checks `dedup_rec.is_duplicate` and `dedup_rec.duplicate_of`. The `Path` is only useful for debugging — you can inspect `self._seen_sha` to trace "this sha was first seen in file X." Could this be removed to save memory? Yes — but it's kept as a debugging aid.

### Q2 — Level 2 fires on address match. Is address set membership transitive?

File A has address `0x111`. File B has addresses `0x111` and `0x222`. File C has address `0x222`.

- File A processed first: `_seen_addr = {0x111: sha_A}`
- File B: Level 2 fires → `0x111` in `_seen_addr` → duplicate of A. **But** `0x222` is NOT registered (the function returns early on Level 2 match before line 88-89).
- File C: `0x222` NOT in `_seen_addr` → Level 2 misses → passes to Level 3.

**Result:** C is NOT caught by Level 2, even though it shares an address with B (which shares an address with A). Address transitivity is BROKEN by the early return.

Is this a bug? Intentional design choice? What would happen if we registered addresses BEFORE checking them?

### Q3 — The `_ADDRESS_RE` pattern catches `0x` + 40 hex characters. What about checksummed addresses (mixed-case)?

The regex is `r'0x[0-9a-fA-F]{40}'` — it matches both lowercase and checksummed (mixed-case) addresses. A checksummed address `0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B` matches. A lowercase version `0xab5801a7d398351b8be11c439e05c5b3259aec9b` also matches. But the SHA-256 of the content differs between the checksummed and lowercase versions. Does Level 2 catch them as duplicates?

### Q4 — `_normalize_for_dedup` does NOT strip string literals. Could a string literal containing `//` or `/*` cause false negatives?

Consider:
```
string memory x = "this contains /* a comment-like string */";
```

The `_BLOCK_CMT` regex `r'/\*.*?\*/'` with `re.DOTALL` would match this and strip it, turning it into:
```
string memory x = "this contains ";
```

This changes the normalized content. Two files where one has `/* ... */` inside a string and the other doesn't would have DIFFERENT normalized hashes, even if the code is otherwise identical. Is this a real problem, and how would you fix it?
