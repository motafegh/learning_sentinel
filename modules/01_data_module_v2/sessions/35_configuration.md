# Session 35 — Configuration (config.yaml)

**Date:** 2026-06-14
**Session number:** 35 of 36
**Mode:** Understand (config.yaml 390L)
**Estimated study time:** 20 min teaching + 10 min Q&A
**Status:** 🟢 Complete

---

## §0.5 What You Already Know

- **Every session so far** references config.yaml values
- **Per-source settings**: tier, connector, pin, crosswalk
- **Pipeline thresholds**: dedup 0.85, cap 3:1, FP fail 0.30, etc.

---

## §1 Top-Level Sections

```
pipeline:                 # global thresholds shared across stages
sources_critical_path:    # 6 sources enabled for Run 11
sources_additive:         # 14 sources deferred to v2.1
sources_dropped:          # 2 sources excluded permanently
deferred_sources:         # 1 source pending v2 baseline results
```

---

## §2 pipeline: — The Threshold Hub (lines 32–96)

| Key | Value | Where Used | What It Controls |
|---|---|---|---|
| `schema_version` | `"v9"` | graph_schema.py | Must match FEATURE_SCHEMA_VERSION |
| `export_shard_size` | `5000` | export writers | Contracts per .pt shard |
| `dedup.ast_similarity_threshold` | `0.85` | deduplicator, dedup_enforcer, overlap_detector | Near-dup detection |
| `verification.fail_threshold` | `0.30` | gate | Tool confidence below → FAIL |
| `verification.tool_corroboration_min` | `1` | tool_validator | Min tools agreeing |
| `verification.negative_tool_hit_threshold` | `0.05` | negative_checker | >5% → WARN, >10% → FAIL |
| `negative.positive_ratio_max` | `3.0` | nonvulnerable_cap | NonVulnerable cap ratio |
| `analysis.complexity_proxy_risk.sigma_threshold` | `1.5` | feature_dist | >1.5σ mean diff → HIGH RISK |
| `analysis.cooccurrence.flag_threshold` | `0.50` | cooccurrence | P(B|A) > 0.50 → flagged |
| `analysis.drift.ks_pvalue_warn` | `0.01` | drift_monitor | KS p < 0.01 → WARNING |

### Min Viable Corpus Gate (lines 66–72)

```yaml
min_viable_corpus:
  total_contracts_min: 4000
  per_class_positive_min_major: 300    # Reentrancy, DoS, IntegerUO
  per_class_positive_min_minor: 100    # other 7 classes
  call_to_unknown_min: 300             # below this → merge rule triggers
  smartbugs_curated_recall_min: 0.90   # semantic checker ground truth
  forge_agreement_min: 0.85            # FORGE gate
```

If the corpus falls below these thresholds at any stage, the pipeline stops with "NOT VIABLE" rather than shipping a bad dataset.

---

## §3 Sources Critical Path (lines 99–223) — 6 Sources

| Source | Tier | Connector | Status | Key Detail |
|---|---|---|---|---|
| DeFiHackLabs | T1 gold | git | **disabled** | Foundry project, forge-std can't compile |
| SolidiFI | T1 gold | git | enabled | 9,369 injected bugs, 100% ground truth |
| DIVE | T1 gold | manual | enabled | 22,330 contracts, multi-label |
| SmartBugs Curated | T3 structural | manual | enabled | 143 hand-labeled, recall ground-truth |
| Web3Bugs | T1 gold | git | enabled | ~3,500 contest-verified bugs |
| DISL | T4 bronze | etherscan | enabled | 514K unlabeled, NonVulnerable only |

**DIVE's complexity** (lines 135–186): DIVE has 14 config fields — the most of any source. Labels CSV, folderization settings, class mapping, staging path. It's the most labor-intensive source to configure.

**DeFiHackLabs deferred** (lines 101–120): A detailed comment (19 lines!) explains WHY it's disabled — Foundry's forge-std dependency means standalone solc can't compile the PoCs. Only 23/738 processed.

---

## §4 Sources Additive (lines 226–368) — 14 Sources

All disabled pending Stage 3 completion. Includes:
- **Bastet** (T1 gold) — replaces Code4rena scraper (legal risk)
- **FORGE** (T1 gold) — requires 50-entry agreement test >85%
- **SmartBugs Wild** (T3) — 47K contracts, 97% FP warning
- **Slither-audited** (T3) — 467K Slither-labeled
- **Zenodo CLEAR** (T4) — ICSE 2025 dataset

---

## §5 Dropped + Deferred (lines 370–385)

| Source | Reason |
|---|---|
| ReentrancyStudy | 230K single-class → BCCC imbalance |
| Code4rena scraper | Legal risk (Bastet replaces) |
| BCCC | 89.4% Reentrancy FP, pending v2 baseline |

---

## §0.6 Exercises (Q1–Q4)

**Q1: What happens if `pipeline.schema_version` doesn't match `graph_schema.FEATURE_SCHEMA_VERSION`?**

The orchestrator checks this on startup. Mismatch → hard error: "schema_version v8 does not match graph_schema v9". This prevents silent feature drift where the config says one version but the code implements another.

**Q2: DeFiHackLabs has `enabled: false` with a 19-line explanation. Why isn't it just deleted?**

The comment serves as institutional memory. If a new engineer asks "why isn't DeFiHackLabs in v2?", they find the answer in config.yaml (not in Slack history). The config IS the decision log.

**Q3: SmartBugs Curated is T3 structural with only 143 contracts. Why is it critical-path?**

Those 143 contracts are hand-labeled ground truth. The `smartbugs_curated_recall_min: 0.90` means the semantic checker must retain ≥90% of these — they're the quality anchor. Small but high-value.

**Q4: DISL uses `connector: etherscan` with no URL. How does Etherscan ingestion work without a URL?**

The Etherscan connector generates the URL from the API key (in env vars) and a list of verified contract addresses. The `url` field is unused for etherscan because scanning is address-driven, not URL-driven. This is a design gap — the `url: ""` is misleading.
