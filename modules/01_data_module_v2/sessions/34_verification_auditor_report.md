# Session 34 — Verification: Class Auditor + Report Generator

**Date:** 2026-06-14
**Session number:** 34 of 36
**Mode:** Understand (class_auditor 189L, report_generator 272L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Verification gate** (Session 23): VERIFIED/PROVISIONAL/BEST-EFFORT/FAIL per class
- **Semantic checker** (Session 24): per-class pass rate from structural graph features
- **FP estimator** (Session 24): detector-based FP rate estimation
- **Negative checker** (Session 24): missed-label detection on non-vulnerable contracts
- **Co-occurrence flagging** (Session 27): merger flags DoS↔Reentrancy from T3/T4 sources

---

## §1 Class Auditor (189 LOC) — The First Verification Step

Runs directly on merged labels — no Slither, no graphs needed.

### Per-Class Stats (lines 34–44)

```python
@dataclass
class ClassStats:
    class_name: str
    total_positives: int
    total_contracts: int
    by_source: dict[str, int]       # positive count per source
    by_tier: dict[str, int]         # positive count per tier
    prevalence: float               # total_positives / total_contracts
```

### Co-occurrence Matrix (lines 48–56)

```python
@dataclass
class CoOccurrencePair:
    class_a, class_b: str
    count: int              # both positive
    count_a: int            # class_a positive
    rate: float             # count / count_a (P(b=1 | a=1))
    flagged: bool           # rate > CO_OCCUR_FLAG_THRESHOLD (0.50)
```

The class auditor's co-occurrence matrix is the **first pass** at detecting BCCC-style noise. The merger flags at the contract level; the auditor flags at the corpus level. Two independent mechanisms catching the same pattern.

### Audit Result (lines 59–80)

```python
@dataclass
class AuditResult:
    per_class: dict[str, ClassStats]
    co_occurrence: list[CoOccurrencePair]
    flagged_pairs: list[CoOccurrencePair]
    total_contracts: int
    duration_s: float

    def summary_lines(self) -> list[str]:
        # Human-readable report printed to CLI
```

---

## §2 Report Generator (272 LOC) — The Unified View

Aggregates ALL verification outputs into one `verification_report.md`:

```python
@dataclass
class VerificationReport:
    audit: AuditResult
    semantic: SemanticCheckResult
    gate: GateResult
    tool_validation: Optional[ToolValidationResult]
    fp_estimation: Optional[FPEstimationResult]
    negative_check: Optional[NonVulnResult]
    corpus_tag: str
```

### Report Sections

```
# SENTINEL v2 Verification Report
**Corpus:** v2.0.0  **Total contracts:** 22,356  **Gate:** PASS ✓

---

## 1. Per-Class Gate
| Class | Verdict | Semantic | Coverage | Best tier | Tool | FP | Co-occur |
|---|---|---|---|---|---|---|---|
| Reentrancy | ✅ VERIFIED | 95% | 88% | T0 | 92% | 5% | |
| IntegerUO | ⚠️ PROVISIONAL | 72% | 80% | T1 | 85% | 18% | |
| AccessControl | 🔶 BEST_EFFORT | 55% | 75% | T2 | — | — | |
| TimestampDependency | ❌ FAIL | 40% | 60% | T3 | — | 45% | ⚠ |

## 2. Per-Class Corpus Stats
| Class | Positives | Prevalence | T0 | T1 | T2 | T3 | T4 |
|---|---|---|---|---|---|---|---|
| Reentrancy | 450 | 2.0% | 200 | 100 | 50 | 50 | 50 |

## 3. Co-occurrence Flags
| A | B | P(B|A) | Flagged |
|---|---|---|---|
| DoS | Reentrancy | 42% | Yes (T4 source) |

## 4. Source Overlap
| Source A | Source B | Exact J | Near J |
|---|---|---|---|
| solidifi | dive | 0.35 | 0.50 |

## 5. Negative Checker
**Status:** WARN — 5% of non-labeled contracts have detector firings.

## 6. FP Estimation
| Class | FP Rate | Negatives Checked | Status |
|---|---|---|---|
| Reentrancy | 5% | 1,200 | OK |
| TimestampDependency | 45% | 800 | FAIL (>30%) |
```

---

## §3 How the Report Is Used

**Two audiences:**
1. **Data team** (daily): Which classes need re-labeling? Are sources stale?
2. **Model team** (per version): Is the dataset safe to train on? Which classes will underperform?

The `corpus_tag` field links the report to a specific dataset version. Reports are stored in `data/analysis/<run_id>/verification_report.md`.

---

## §0.6 Exercises (Q1–Q4)

**Q1: The report shows TimestampDependency with FAIL verdict, semantic_pass_rate=40%, fp_rate=45%. What should the data team do?**

Two independent signals agree: the labels are unreliable. Action: (1) Review the T3/T4 source that contributes most TimestampDependency positives. (2) Either remove or re-label from a higher-tier source. (3) Re-run verification. The class is blocked from export until the issue is fixed.

**Q2: The report's Gate section shows Co-occurrence flagged for DoS↔Reentrancy. Is this from the merger or the auditor?**

The report aggregates BOTH. The `co_occurrence_flagged` column in the table likely references the merger's per-contract flagging. The separate §3 Co-occurrence Flags section references the auditor's corpus-level matrix. Both are shown for comparison.

**Q3: Why does the report have both `Tool` and `FP` columns? Aren't they the same?**

No. `Tool` = tool_validator's agreement rate (Slither matches label for labeled contracts). `FP` = fp_estimator's rate (Slither fires on non-labeled contracts, suggesting false positives). They measure different things: tool agreement on POSITIVES vs. tool activity on NEGATIVES.

**Q4: The Gate shows `PASS ✓` but TimestampDependency is FAIL. Can the dataset still be exported?**

Yes — gate-level pass means the corpus AS A WHOLE is acceptable. Per-class FAIL means that specific class is blocked from export. The export pipeline has a class filter that excludes FAIL classes. The model trains on the remaining classes only.
