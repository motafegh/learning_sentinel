# Session 26 — Label Parsers (DIVE, SmartBugs Curated, SolidiFI)

**Date:** 2026-06-14
**Session number:** 26 of 36
**Mode:** Understand (dive 167L, smartbugs_curated 165L, solidifi 162L)
**Estimated study time:** 25 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Label format** (Session 22): per-contract `.labels.json` with `{... classes: {Reentrancy: {value: 1, tier: T0}, ...}}`
- **Schema** (Session 22): 9 vulnerability classes + NonVulnerable
- **Label flow**: source `original_path` → folder name → crosswalk YAML → canonical class → `.labels.json`
- **Ingestion** (Session 06): raw repos are fetched into `data/raw/<source>/`

---

## §1 All Three Parsers Share a Pattern

```
1. Read meta.json for each preprocessed contract (from data/preprocessed/<source>/)
2. Extract folder name from original_path field
3. Look up folder → canonical class via crosswalk YAML
4. Write data/labels/<source>/<sha256>.labels.json
```

### Crosswalk files

Each source has a YAML crosswalk at `labeling/crosswalks/<source>.yaml`:

| Source | Crosswalk mapping |
|---|---|
| SolidiFI | `Re-entrancy` → `Reentrancy`, `Timestamp` → `TimestampDependency`, ... |
| SmartBugs Curated | `reentrancy` → `Reentrancy`, `arithmetic` → `IntegerUO`, ... |
| DIVE | `Reentrancy` → `Reentrancy` (same name), `Bad Randomness` → *dropped* |

---

## §2 SolidiFI Parser (162 LOC)

```python
def _extract_folder(original_path):
    # "repo/buggy_contracts/Re-entrancy/buggy_3.sol" → "Re-entrancy"
    parts = Path(original_path).parts
    if len(parts) >= 3 and parts[1] == "buggy_contracts":
        return parts[2]
    return None
```

**Key quirk:** `parts[1]` is `buggy_contracts` (one level deeper than SmartBugs). The folder names are hyphenated (e.g., `Re-entrancy`) and mapped through crosswalk to canonical names (e.g., `Reentrancy`).

**Single-label assumption:** One contract → one vulnerability category. The category folder is a direct injection.

---

## §3 SmartBugs Curated Parser (165 LOC)

```python
def _extract_folder(original_path):
    # "repo/reentrancy/reentrancy_1.sol" → "reentrancy"
    parts = Path(original_path).parts
    if len(parts) >= 3 and parts[0] == "repo":
        return parts[1]
    return None
```

**Key quirk:** `parts[0]` is `repo` (one level shallower than SolidiFI). The crosswalk maps lowercase hyphenated folder names (`re-entrancy`) to canonical classes.

---

## §4 DIVE Parser (167 LOC) — The Multi-Label One

Unlike the other two, DIVE contracts can be POSITIVE for MULTIPLE classes. The same `.sol` file appears in multiple vulnerability folders.

```python
def _build_folder_index(raw_repo, class_map):
    """Scan every mapped folder for .sol files.
    Same filename in Reentrancy/ and Arithmetic/ → both classes are positive.
    """
    for folder_name, canonical_class in class_map.items():
        for sol in (raw_repo / folder_name).glob("*.sol"):
            index.setdefault(sol.name, set()).add(canonical_class)
```

**Folder membership = positive labels.** Absence from ALL folders = NonVulnerable. The `__source__/` directory has all contracts; vulnerability folders have symlinks/copies of the subset that is vulnerable.

**Bad Randomness dropped:** DIVE has a "Bad Randomness" folder with no canonical equivalent. Those contracts get `NonVulnerable` label (conservative — they may still be vulnerable in other categories).

**NonVulnerable detection asymmetry:** If a contract is in `__source__/` but NOT in any vulnerability folder, it's NonVulnerable. However, NonVulnerable contracts never appear in the vulnerability folders — so the absence AS well as presence signals the label.

---

## §0.6 Exercises (Q1–Q4)

**Q1: A SolidiFI contract has `original_path = "repo/buggy_contracts/Access_Control/buggy_12.sol"`. The crosswalk maps `Access_Control` to `AccessControl`. Which class does the parser assign?**

`AccessControl` — one class per contract. The `_build_labels_json` sets that class to `{"value": 1, "tier": "T0"}` and all others to `{"value": 0}`.

**Q2: A DIVE contract appears in both `Reentrancy/` and `Arithmetic/` folders. How many `.labels.json` files are written?**

One. The parser builds one `.labels.json` with both `Reentrancy` and `IntegerUO` (canonical for Arithmetic) set to `value: 1`. This is the multi-label design — the merger later reconciles across sources.

**Q3: The SmartBugs Curated parser finds `original_path = "repo/reentrancy/reentrancy_1.sol"`. The crosswalk has no entry for `reentrancy`. What happens?**

The parser logs a warning, increments `labels_failed`, and does NOT write a `.labels.json`. This contract is skipped. The merger then won't have labels for this contract from this source.

**Q4: DIVE's `_build_folder_index` returns a dict keyed by filename. What happens if two contracts have the same filename but different sha256s?**

The index would merge them (same key = same set of canonical classes). But this doesn't cause label contamination because the labels are written per-sha256 (each contract is identified by sha256, not filename). The folder index is used only to determine WHICH classes are positive; the sha256 comes from the per-contract meta.json read.
