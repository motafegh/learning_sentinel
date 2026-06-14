# Session 23 — Verification: Runner + Validator + Gate

**Date:** 2026-06-14
**Session number:** 23 of 36
**Mode:** Understand (Slither runner 250L, tool validator 273L, gate 286L)
**Estimated study time:** 30 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0.5 What You Already Know

- **Labeled contracts** (Session 22): merged `.labels.json` with tier, classes, provenance
- **Graph features** (Session 17): CEI path, block globals, unchecked detection — used by semantic checker
- **Slither** (Session 16): used in graph extraction; now used again for detector-based verification

---

## §1 Slither Runner — Shared Cache Layer (250 LOC)

The shared module that runs Slither detectors and caches results.

### `CLASS_TO_DETECTORS` — Class → Slither Detector Mapping (lines 26–45)

```python
CLASS_TO_DETECTORS = {
    "Reentrancy":      ["reentrancy-eth", "reentrancy-no-eth", "reentrancy-benign", "reentrancy-events"],
    "CallToUnknown":   ["low-level-calls", "controlled-delegatecall", "delegatecall-loop"],
    "Timestamp":       ["timestamp"],
    "IntegerUO":       [],           # no dedicated Slither detector in v0.10
    "MishandledException": ["unchecked-send", "unchecked-lowlevel", "unchecked-transfer", "return-bomb"],
    "UnusedReturn":    ["unused-return"],
    "ExternalBug":     ["arbitrary-send-eth", "low-level-calls", "tx-origin", "controlled-delegatecall"],
    "DenialOfService": ["calls-loop", "costly-loop", "msg-value-loop"],
    "GasException":    ["calls-loop", "costly-loop", "low-level-calls"],
    "TransactionOrderDependence": ["tx-origin", "controlled-delegatecall"],
}
```

**Detector assignment principles:**
- Slither 0.10 removed `integer-overflow` (Solidity 0.8+ handles overflow natively) → `IntegerUO` has zero detectors
- A single detector can map to multiple classes (e.g., `low-level-calls` → CallToUnknown AND ExternalBug)
- Detectors are used CORROBORATIVELY, not authoritatively

### `run_on_contract(sol_path) → SlitherFindings`

Returns a dataclass with:
- `sha256`: content hash
- `detectors_fired`: list of detector names that flagged the contract
- `detector_map`: class → list of detectors that fired for that class
- `all_results_raw`: raw Slither output for debugging
- `error`: error string if Slither crashed

### Content-Addressed Cache (lines 6–7)

```
Cache: data_dir/slither_cache/<source>/<sha256>.slither.json
Key:   sha256 of the .sol content
```

Same sha256 as preprocessed output. Cache hit = free re-run. First run: ~5–30s per contract.

---

## §2 Tool Validator — Per-Class Agreement (273 LOC)

For each labeled positive, runs Slither and reports whether its detectors agree.

### `AgreementVerdict` (lines 42–49)

| Verdict | Meaning |
|---|---|
| `AGREE` | At least one class detector fired |
| `DISAGREE` | Slither ran cleanly; no class detector fired |
| `NO_DETECTOR` | Class has no Slither detector (IntegerUO) |
| `SKIP` | Preprocessed .sol not found |
| `ERROR` | Slither run errored |

### Per-Class Stats (lines 64–80)

```python
ClassAgreementStats:
    positives_total   # labeled positives seen
    agree             # Slither agreed
    disagree          # Slither disagreed
    no_detector       # class has no detector
    skipped           # .sol not found
    errored           # Slither crash
    checkable         # agree + disagree (denominator for agreement_rate)
```

`agreement_rate = agree / checkable` — the fraction of Slither-checkable positives where Slither also flagged the same class. Low agreement is suspicious but not conclusive (Slither has known FPs and FNs).

### Run-Time Contract (lines 18–22)

- 7K positives × ~5s = ~10 hours first run
- Cache makes repeated runs ~instant
- `limit_per_class=N` for smoke tests
- BCCC regression test defaults to 50/class

---

## §3 Verification Gate — Go/No-Go (286 LOC)

Combines 5 inputs → per-class verdict:

```
Inputs:
  ├── class_auditor        → AuditResult (co-occurrence flags)
  ├── semantic_checker     → SemanticCheckResult (pass_rate per class)
  ├── tool_validator       → ToolValidationResult (agreement_rate)
  ├── fp_estimator         → FPEstimationResult (fp_rate)
  └── negative_checker     → NonVulnResult (corpus-level status)

Output: per-class Verdict + summary
```

### Verdict Rules (lines 7–28)

```
VERIFIED       semantic pass_rate > 90%  AND no co-occurrence flag
PROVISIONAL    semantic pass_rate 60–90%  OR  T0/T1 without graph reps
BEST-EFFORT    semantic pass_rate 30–60%  OR  NOT_EXTRACTABLE class with T2+
FAIL           semantic pass_rate < 30%
               OR  co-occurrence flag on high-noise source
               OR  fp_rate > 30%
               OR  tool_agreement < 30% AND co-flag
```

**T0 override** (lines 16–17): SolidiFi (T0, injection-verified) with zero semantic failures → VERIFIED regardless of rep coverage. Ground truth by construction.

**T2 fallback** (lines 19–20): DIVE (T2, curated) with no semantic checks → PROVISIONAL. Trusted curation but unverified at AST level.

**Hard gate:** Any FAIL class blocks downstream export.
**Soft gate:** PROVISIONAL / BEST-EFFORT exports with warnings.

---

## §0.6 Exercises (Q1–Q4)

**Q1: `IntegerUO` has zero Slither detectors. What verdict does it get from the gate?**

`IntegerUO` gets `NO_DETECTOR` from the tool validator (checkable=0, agreement_rate=None). The gate doesn't receive a tool_validator signal for this class. The verdict depends entirely on semantic_checker pass_rate and class_auditor co-occurrence flags. If semantic pass_rate > 90%, it gets VERIFIED.

**Q2: The Slither cache uses sha256 of the .sol content. What happens when a new extraction produces a different .pt file for the same sha256?**

The `.pt` file content doesn't affect the Slither cache. The cache key is `(sha256_of_sol, )` — only the Solidity source matters. Re-extracting graphs doesn't invalidate Slither results. This is correct because Slither runs on the `.sol` file, not the graph.

**Q3: A T0 source (solidifi) labels a contract Reentrancy. Semantic pass_rate is 100%, no co-occurrence flags. What's the gate verdict?**

VERIFIED. The T0 override at lines 16–17 applies: "T0 with no semantic failures → VERIFIED regardless of rep coverage." The tool_validator agreement_rate is irrelevant for VERIFIED (if semantic pass_rate > 90% AND no co-flag, it's VERIFIED even if Slither disagrees).

**Q4: What happens when the gate produces a FAIL verdict?**

A FAIL class blocks downstream export. The gate report must be reviewed by a human, who either fixes the underlying data (removes noisy labels, re-runs verification) or documents a deferral decision (ADR). The pipeline cannot proceed to splitting/export with any FAIL class.
