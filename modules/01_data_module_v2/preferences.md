# Preferences — `sentinel-data` Module

> **Inherits from global `../../preferences.md` (P1–P20).** All global rules
> apply. This file adds module-specific rules that are unique to learning
> `sentinel-data`. Module rules may ADD constraints; they never override
> global rules. If they appear to conflict, the global rule wins.

---

## Module-Specific Rules

### P101 — Always Cross-Check Thin-Adapter Targets Against `ml/src/`

**Rule:** when learning any file in `sentinel_data/representation/`, the
actual code lives in `ml/src/preprocessing/` and `ml/src/data_extraction/`.
The representation files are ~10-line re-export wrappers. Reading the wrapper
without reading the target gives you a false sense of understanding.

**Applies to:** `sentinel_data/representation/{graph_schema,graph_extractor,tokenizer}.py`

**Format:** when teaching a thin-adapter file, paste the 3–10 lines of the
wrapper, then **navigate to the target** in `ml/src/` and teach from there.

**Rationale:** the byte-identical regression test
(`tests/test_representation/test_byte_identical_regression.py`) enforces
`from sentinel_data.X import Y is from ml.X import Y`. The wrapper is
a packaging detail; the logic is in `ml/`.

---

### P102 — The Two-Taxonomy Divergence Is the First Concept Taught

**Rule:** Session 1 of the data_module roadmap MUST open with §6.1 of
`reference.md` (the two-taxonomy divergence). This must be internalized
**before** any code is read, because:

1. It explains why `graph_schema.CLASS_NAMES` ≠ `taxonomy.yaml:class_names()`
2. It explains why any index-aligned work must use the representation order
3. It explains why the v2 build still references v1 column-order contracts in
   `data/processed/multilabel_index.csv`

**Format:** the divergence must be the **first concept in the very first
chunk** of the module, before the architecture overview. The user must be
able to answer the question: *"If I want my new class at index 9, do I edit
graph_schema.py or taxonomy.yaml — and what else must I update?"* before
moving on.

---

### P103 — Schema Constants Are Facts, Not Memorization

**Rule:** the v9 schema constants (FEATURE_SCHEMA_VERSION, NODE_FEATURE_DIM,
NUM_NODE_TYPES, NUM_EDGE_TYPES, NUM_CLASSES, EXTRACTOR_VERSION, _MAX_TYPE_ID)
go in the **"3 things to lock in"** section of every representation-stage
chunk. They are facts, not syntax to memorize.

**Why:** these constants are the contract between Stage 2 and the model
checkpoint. Changing any of them invalidates the cache and all existing
checkpoints. They appear in audit discussions, bug reports, and config.yaml.
They are looked up, not recalled.

**Source:** `reference.md` §6.3.

---

### P104 — When the Code Says "STUB", That Is the Truth

**Rule:** the following are STUBS as of 2026-06-12 — do not pretend they
are implemented:

- `sentinel_data/export/` (10-line `__init__.py` + scaffolding only — Stage 7
  is not wired)
- `sentinel_data/cli.py:223-229` (label subcommand stub — merger runs from
  Python)
- `sentinel_data/representation/{call_graph,opcode_extractor,pdg_builder}.py`
  (35-line `NotImplementedError` placeholders — deferred to v3.1)
- `sentinel_data/preprocessing/deduplicator.py` AST near-dup (3rd dedup
  level) — only SHA-256 and address dedup are real

**Format:** when teaching a stub, mark it explicitly:

```
# Awareness only — STUB. Deferred to <target version>.
```

**Do not** try to teach how the stub "would work" if implemented. If the
target version is v2.1 or v3.1, the design may change before it lands.

---

### P105 — The Pipeline Order Is the Learning Order

**Rule:** teach in pipeline order. The DVC DAG is:
`ingest → preprocess → represent → label → verify → split → register → analyze → export`.

**Why:** every stage reads the previous stage's well-known output dir. If you
teach verification before labeling, the user has no idea what
`data/labels/merged/*.labels.json` is. If you teach splitting before
verification, the user has no idea what `data/verification/contracts_clean.csv`
is.

**Exception:** Session 1 is the orientation (architecture + config) before
any code. This is the only deviation from pipeline order.

---

### P106 — Audit the Two-Taxonomy Divergence Aggressively

**Rule:** when reading any file that uses class indices (not class names),
**audit inline** whether the author knew about the two-taxonomy divergence.
Common mistakes to flag:

- `if pred.argmax() == 5` — what class is 5? Representation or labeling order?
- Reading from `taxonomy.yaml` but indexing against `graph_schema.CLASS_NAMES`
- Hardcoding `NUM_CLASSES = 9` instead of `10` (the NonVulnerable slot is
  easy to miss in the labeling order)

**Format:** raise `[AUDIT]` flags with the divergence explicitly named:

```
[AUDIT] — uses class index 5 (Reentrancy in representation order, but
          DoS in labeling order). The two orders diverge at index 5
          AND index 8 (NonVulnerable ↔ TransactionOrderDependence swap).
          See reference.md §6.1.
```

---

### P107 — When Reading a Thin-Adapter, Read the Target Too

**Rule:** per P101, but more specific — when reading
`sentinel_data/representation/graph_schema.py`, **also read**
`ml/src/preprocessing/graph_schema.py` in the same chunk. They are
inseparable. The byte-identical test guarantees they are the same object,
but understanding the schema (NODE_TYPES, EDGE_TYPES, FEATURE_NAMES,
VISIBILITY_MAP, STRUCTURAL_PREFIX_TYPES) requires reading the actual
implementation, not the re-export statement.

**Format:** present them side-by-side, with the wrapper called out as a
packaging concern.

---

### P108 — Confidence Tier System (T0–T4) Is a First-Class Concept

**Rule:** introduce the confidence tier system (per
`docs/architecture.md:71-81`) in Session 5 (ingestion overview). Every
downstream stage (labeling merger, verification gate, analysis drift
monitor) consumes `tier` as a first-class signal.

**Format:** the tier table is the **first thing taught in the labeling
session**, before the merger code. The user must internalize:

- T0 = on-chain exploit proof (DeFi Hacks REKT)
- T1 = gold (human-curated or mathematically certain)
- T2 = silver (expert auditors or 3/5 tool majority)
- T3 = bronze (tool-generated, conservative threshold)
- T4 = unlabeled (pretraining / NonVulnerable pool only)

**Why:** "tool-agreement is corroborative, not authoritative. Human audit
(T0/T1) always overrides tool labels (T2/T3)." (from
`docs/architecture.md:81`)

---

### P109 — Slither Detectors Map Is the Source of Truth for "What Can Be Detected"

**Rule:** when teaching the verification stage, open with
`slither_runner.CLASS_TO_DETECTORS` (lines 26–45). The 10-class map to
Slither detector names is the contract between the data layer and the
verification layer.

**Critical fact:** IntegerUO has **NO Slither detector in v0.10** because
Solidity 0.8+ handles it natively. This means `tool_validator`, `fp_estimator`,
and `negative_checker` all skip IntegerUO. Only `semantic_checker` (via
`feat[11] unchecked_block` or pre-0.8 solc) can corroborate.

**Source:** `verification/README.md:55-65`.

---

### P110 — Defer to Roadmap Order, Not Source Tree Order

**Rule:** the `sentinel_data/` package organizes files by **subpackage**
(`ingestion/`, `preprocessing/`, etc.). The **roadmap** in `reference.md §9`
organizes sessions by **teaching unit** (which may combine multiple files
across subpackages, or split one subpackage into multiple sessions).

**When in doubt:** follow the roadmap. The roadmap accounts for file size,
conceptual grouping, and prerequisite dependencies. The source tree is the
filesystem layout, not the curriculum.

---

## See also

- Global: `../../preferences.md` (P1–P20)
- Global spec protocol: `../../README.md#the-spec-file-update-protocol`
- This module's reference: `./reference.md`
- This module's mission: `./mission.md`
- This module's audit log: `./audit_flags.md`
- This module's session log: `./session_log.md`
- This module's session study docs: `./sessions/` (one per session; this is the persisted study material)
- This module's learning records: `./learning-records/` (created lazily)
- Teach skill format reference: `~/.agents/skills/teach/` (mission + learning-record formats)
