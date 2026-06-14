# Session 22 — Labeling System (`labeling/`)

**Date:** 2026-06-14
**Session number:** 22 of 36
**Mode:** Understand (918 LOC — parsers, merger, gate, schema)
**Estimated study time:** 35 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **10-class vocabulary** (Session 15): CallToUnknown, DenialOfService, ExternalBug, GasException, IntegerUO, MishandledException, Reentrancy, Timestamp, TransactionOrderDependence, UnusedReturn
- **Preprocessed output** (Sessions 08–14): `<sha256>.sol` + `meta.json` per contract
- **DIVE folder structure** (Session 07): `__source__/<id>.sol` (flat) + class folders with symlinks

---

## §1 Architecture — 4 Components

```
preprocessed/                         (Stage 1 output)
  <source>/
    <sha256>.meta.json

labeling/parsers/                     (Stage 3.1–3.7: per-source parsers)
  dive.py                             ← folder membership → canonical class names
  smartbugs_curated.py                ← hardcoded per-contract label map
  solidifi.py                         ← CSV lookup + class mapping

  ↓ per-source .labels.json

labeling/merger.py                    (Stage 3.10: merge + conflict resolution)
  ↓ merged .labels.json

labeling/gate.py                      (Stage 3.11: minimum viable corpus check)
  ↓ pass/fail report
```

---

## §2 Parsers (dive 167 LOC, smartbugs_curated 165, solidifi 162)

Each parser reads source-specific label data and writes per-contract `.labels.json`:

```json
{
    "sha256": "...",
    "source": "dive",
    "tier": "T2",
    "classes": ["Reentrancy"],
    "confidence": 1.0
}
```

### DIVE Parser — Folder Membership (167 lines)

**Label mechanism:** DIVE stores all contracts in `__source__/<N>.sol`. The same file appears in vulnerability folders (e.g., `Reentrancy/<N>.sol`). Folder membership = positive labels.

**Crosswalk YAML:** Maps folder names to canonical class names:
```yaml
# labeling/crosswalks/dive.yaml
Reentrancy: Reentrancy
Arithmetic: IntegerUO
Timestamp_Dependence: Timestamp
Bad_Randomness: null  # dropped — no canonical equivalent
```

**`_build_folder_index` (lines 54–69):** Scans each mapped folder for `.sol` files, builds `filename → frozenset[canonical_class]` index. Files in no vulnerability folder → NonVulnerable.

### SmartBugs Curated Parser (165 lines)

Uses a hardcoded per-contract label map from the original SmartBugs dataset. Each contract has a known ground-truth label. No crosswalk needed.

### SolidiFi Parser (162 lines)

Reads a CSV with contract-address → vulnerability-class mappings. Applies the crosswalk to convert source-specific class names to canonical 10-class vocabulary. Contracts with unmapped classes are skipped.

---

## §3 Merger — Multi-Source Conflict Resolution (240 lines)

```
Input:  data/labels/<source>/*.labels.json   (per-source)
Output: data/labels/merged/*.labels.json      (canonical)

Process:
  1. Group same-sha256 contracts across sources
  2. Apply tier precedence: T0 > T1 > T2 > T3 > T4
  3. Within same tier: positive wins over negative
  4. Detect DoS+Reentrancy co-occurrence noise
  5. Write merged .labels.json
```

### Confidence Tiers

| Tier | Sources | Meaning |
|---|---|---|
| T0 | solidefi, defihacklabs | Manual audit — ground truth |
| T1 | smartbugs_curated, web3bugs | Curated dataset — verified |
| T2 | dive | Folder membership — likely correct |
| T3 | disl | Automated tool output |
| T4 | (future) | ML prediction — lowest confidence |

### Conflict Resolution

Two sources label the same contract:

| Source A | Source B | Result |
|---|---|---|
| T0: Reentrancy | T2: Reentrancy | T0 wins → Reentrancy |
| T0: Reentrancy | T2: Timestamp | T0 wins → Reentrancy |
| T2: Reentrancy | T4: [] (nonvuln) | T2 wins → Reentrancy |
| T2: Reentrancy | T2: Timestamp | Both kept → multi-label |

### DoS+Reentrancy Co-occurrence Noise Rule

BCCC had 99% co-occurrence of DoS and Reentrancy labels — meaning the labeling was unreliable (every contract labeled as both). The merger detects this: if a T3/T4 source labels >50% of its contracts with BOTH DoS AND Reentrancy, those contracts are flagged for Stage 4 verification (not silently dropped).

---

## §4 Gate — Minimum Viable Corpus (139 lines)

After merging, the gate checks config thresholds:

| Criterion | Default threshold | Source |
|---|---|---|
| `total_contracts_min` | 5000 | Config: `pipeline.min_viable_corpus` |
| `per_class_positive_min_major` | 200 | Reentrancy, DoS, IntegerUO |
| `per_class_positive_min_minor` | 50 | Other 7 classes |
| `call_to_unknown_min` | 50 | Flags for human review if below |

If any criterion fails, the gate reports `pass=False` with actual vs threshold values. The pipeline CAN proceed with a failing gate if the deferral decision is documented (per ADR).

---

## §5 Label Output Format

```json
{
    "sha256": "abc123...",
    "source": "merged",
    "tier": "T0",
    "classes": ["Reentrancy"],
    "confidence": 1.0,
    "provenance": [
        {"source": "solidifi", "tier": "T0", "classes": ["Reentrancy"]},
        {"source": "dive", "tier": "T2", "classes": ["Reentrancy"]}
    ]
}
```

The `provenance` array traces the label's origin — every source that contributed to the final label, with its tier and class set. This is the audit trail for Stage 4 verification.
