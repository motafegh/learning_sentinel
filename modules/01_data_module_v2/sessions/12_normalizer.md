# Session 12 — Normalizer (`preprocessing/normalizer.py`)

**Date:** 2026-06-14
**Session number:** 12 of 36
**Mode:** Understand (39 lines — 5-regex source cleaning in deterministic order)
**Estimated study time:** 25 min teaching + 10 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **PreprocessingPipeline Step 4** (Session 08): `normalize(flat.content)` receives flattened, compiled, deduplicated content. Output is the final `.sol` text written to `out_dir/<sha256>.sol`. Only the SHA-256 filename survives from the original — all source formatting is normalized away.

---

## §1 The 5 Regex Passes (lines 18–34)

```python
_SPDX_RE    = re.compile(r'^//\s*SPDX-License-Identifier:[^\n]*\n?', re.MULTILINE)
_LINE_CMT   = re.compile(r'//[^\n]*')
_BLOCK_CMT  = re.compile(r'/\*.*?\*/', re.DOTALL)
_MULTI_NL   = re.compile(r'\n{3,}')
_TRAIL_WS   = re.compile(r'[ \t]+$', re.MULTILINE)
```

| Order | Regex | Purpose | Example |
|---|---|---|---|
| 1 | `_SPDX_RE` | Remove `// SPDX-License-Identifier: ...` header | `// SPDX-License-Identifier: MIT` → `""` |
| 2 | `_BLOCK_CMT` | Remove `/* ... */` block comments | `/* @dev safe function */` → `""` |
| 3 | `_LINE_CMT` | Remove `//` line comments | `// this is a comment` → `""` |
| 4 | `_TRAIL_WS` | Remove trailing spaces/tabs on each line | `x = 1;   ` → `x = 1;` |
| 5 | `_MULTI_NL` | Collapse 3+ blank lines to exactly 2 | `\n\n\n\n` → `\n\n` |
| 6 (implied) | `strip() + '\n'` | Remove leading/trailing whitespace, ensure trailing newline | |

### Why SPDX First

SPDX headers look like `// SPDX-License-Identifier: MIT` — a special case of a line comment. If `_LINE_CMT` ran first, it would remove SPDX anyway. But SPDX is done separately because:
- The SPDX regex is more precise (anchored at line start, matches the keyword)
- Future work could extract the license identifier instead of deleting it

### Why Line Comments After Block Comments

If `_LINE_CMT` ran before `_BLOCK_CMT`, a line like:
```solidity
/* some text // still in block comment */
```
The `//` inside the block comment would trigger `_LINE_CMT`, producing:
```solidity
/* some text 
```
Which leaves an unclosed block comment. Running block comments first avoids this — the entire `/* ... */` is removed in one pass.

---

## §2 The `strip() + '\n'` Invariant (line 34)

```python
out = out.strip() + '\n'
```

Every preprocessed file ends with exactly one trailing newline and has no leading/trailing whitespace. This is the **canonical form** guarantee. Downstream consumers (graph extraction, tokenization) never need to handle edge cases like missing trailing newlines or leading blank lines.

---

## §3 Known Limitation — String-Embedded Comments

As in the deduplicator (Session 09), the `_BLOCK_CMT` regex `r'/\*.*?\*/'` with `re.DOTALL` matches inside string literals:
```solidity
string memory x = "this contains /* a non-comment */ string";
```
→ becomes:
```solidity
string memory x = "this contains  string";
```

This changes the string literal content. However, in Solidity this pattern is extremely rare — test strings that happen to contain `/*` are effectively non-existent in the benchmark datasets. Fixing it would require a Solidity-aware parser (antlr4 or tree-sitter), which is deferred to v2.x.

---

## §0.6 Exercises (Q1–Q4)

### Q1 — What if the input source is empty or whitespace-only?

Empty file → `strip()` → `""` → `"" + '\n'` = `"\n"`. `n_lines_before = 0` (empty file has no newlines), `n_lines_after = 1` (just the trailing newline).

Whitespace-only → `strip()` → `""` → `"\n"`. Same result. The pipeline accepts this — a normalized-empty file passes to the segmenter, which won't find any contract names, producing `primary = "Unknown"` and `version_bucket = "legacy"`. The file is still written to output. These degenerate contracts are filtered later during labeling or splitting.

### Q2 — The `_TRAIL_WS` regex `[ \t]+$` uses `$` with `re.MULTILINE`. Does it also match at the end of the string?

Yes. `$` matches both end-of-line (before `\n`) and end-of-string in MULTILINE mode. So trailing whitespace on the final line (which may not have a trailing newline at this point) is also stripped.

### Q3 — `_MULTI_NL` collapses `\n{3,}` to `\n\n`. Why not `\n{2,}` → `\n\n`?

Collapsing `\n\n` → `\n\n` is a no-op. Using `\n{3,}` means single blank lines (paragraph separators) are preserved while 2+ blank lines are collapsed to exactly 1 blank line. The threshold of 3 means "at most one blank line between code blocks."

### Q4 — `n_lines_before` is computed as `source.count('\n') + 1`. For a file with 0 newlines (single line), this gives 1. Correct. For a file ending with `\n` (100 newlines), this gives 101 lines. The last newline is an empty line 101. Is this correct?

Yes — this is the standard Unix line count convention. A 100-line file has newlines at the end of lines 1–100. The 101st count is the trailing empty line after the final `\n`. But `n_lines_after` also uses this convention, so the compression ratio `n_lines_after / n_lines_before` is comparable even if both are inflated by 1.
