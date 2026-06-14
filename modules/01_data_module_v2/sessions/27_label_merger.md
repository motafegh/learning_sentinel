# Session 27 — Label Merger

**Date:** 2026-06-14
**Session number:** 27 of 36
**Mode:** Understand (merger.py 240L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Per-source labels** (Session 26): SolidiFI (T0), DIVE (T2), SmartBugs (T1), etc.
- **Same contract can be in multiple sources** (from ingestion dedup groups)
- **Schema** (Session 22): 9 classes + NonVulnerable, tier precedence T0 > T1 > T2 > T3 > T4

---

## §1 What the Merger Does (lines 1–16)

```
Input:  data/labels/<source>/*.labels.json  (per-source, per-contract)
Output: data/labels/merged/<sha256>.labels.json  (one canonical per contract)
```

**Key tasks:**
1. Detect contracts that appear in multiple sources (same sha256, multiple `.labels.json`)
2. Merge class entries using tier-based conflict resolution (D-3.3)
3. Flag DoS↔Reentrancy co-occurrence from low-confidence sources
4. Write one canonical `.labels.json` per contract

---

## §2 Conflict Resolution Strategy (lines 76–97)

```python
_TIER_ORDER = ["T0", "T1", "T2", "T3", "T4", None]
_SOURCE_PRECEDENCE = ["solidifi", "defihacklabs", "smartbugs_curated",
                       "web3bugs", "dive", "disl"]
_SOURCE_TIER = {
    "solidifi":        "T0",
    "defihacklabs":    "T0",
    "smartbugs_curated": "T1",
    "web3bugs":        "T1",
    "dive":            "T2",
    "disl":            "T4",
}
```

### Step-by-step for a class `C`:

1. Collect all entries for `C` from all sources (each entry = `{value: 0|1, tier: T0-T4, source}`)
2. Separate positives and negatives
3. If any positive exists: pick the one with lowest tier rank (highest confidence). T0 > T1 > ...
4. If NO positives: keep negative with lowest tier rank
5. Merge into one entry: `{value: 1, tier: "T0", source: "solidifi"}`

**Result:** The merged label gets the HIGHEST confidence tier available. If both SolidiFI (T0) and DIVE (T2) label Reentrancy, the merged entry uses T0.

---

## §3 Co-occurrence Flagging (lines 100–140)

```python
CO_OCCUR_NOISE_THRESHOLD = 0.50
_LOW_CONFIDENCE_TIERS = {"T3", "T4"}
```

**Problem:** A T3/T4 source labels both DoS AND Reentrancy on most contracts → artificial co-occurrence pattern that distorts the model.

**Solution:** For each T3/T4 source, compute `CoS_Frac = contracts_with_both / contracts_with_either`. If > 50%, flag those contracts. The flag is informational only — the gate makes the final decision.

**Important asymmetry:** DIVE (T2) at 12% co-occurrence is NOT flagged. That's legitimate multi-label signal. The threshold targets the BCCC 99% co-occurrence pattern from low-confidence sources.

---

## §4 Merger Output Format

```json
{
    "sha256": "abc123...",
    "source": "solidifi",
    "tier": "T0",
    "sources": ["solidifi", "dive"],
    "classes": {
        "Reentrancy": {"value": 1, "tier": "T0", "source": "solidifi"},
        "IntegerUO": {"value": 0, "tier": "T0", "source": "solidifi"},
        "AccessControl": {"value": 1, "tier": "T2", "source": "dive"},
        ...
    },
    "meta": {
        "co_occurrence_flagged": false,
        "n_sources": 2,
        "source_tiers": ["T0", "T2"]
    }
}
```

Note: each class tracks which source it came from (`"source"` field). The top-level `"source"` is the highest-tier source overall.

---

## §0.6 Exercises (Q1–Q4)

**Q1: SolidiFI (T0) labels Reentrancy=1. DIVE (T2) labels Reentrancy=0. What's the merged Reentrancy value?**

1 (positive). The tier precedence rule says: if ANY positive exists, take the highest-confidence positive. T0 > T2, so SolidiFI's positive wins.

**Q2: DISL (T4) labels DoS=1 and Reentrancy=1 on 60% of its contracts. No other source covers these contracts. What happens?**

The co-occurrence checker flags these contracts (60% > 50% threshold). The flag is recorded per contract. The merger outputs merged labels with the `co_occurrence_flagged: true` meta field. The gate (Stage 4) decides whether to block these labels from export.

**Q3: A contract has non-overlapping positive classes from SolidiFI (Reentrancy=1) and DIVE (AccessControl=1). How does the merger handle it?**

Clean merge — no conflict. The merged label has Reentrancy=1 (from SolidiFI, T0) and AccessControl=1 (from DIVE, T2). This is a legitimate multi-label contract.

**Q4: The merger encounters a contract present in 3 sources. How does it determine which source's None-tier entry wins?**

`_TIER_ORDER = ["T0", "T1", "T2", "T3", "T4", None]`. `None` is last (lowest priority). The merged label uses the entry from the source with the lowest tier rank. If multiple sources have the same tier, `_SOURCE_PRECEDENCE` breaks ties (solidifi > defihacklabs > smartbugs_curated > ...).
