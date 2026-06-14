# Index — Sentinel Learning Project

> **Master index of all modules in `~/projects/learning_sentinel/modules/`.**
> Read first → then `README.md` → then `preferences.md` → then pick a module.

---

## Status legend

- ✅ **Set up** — 5 files + 1 lazy sub-dir in place, ready to start Session 1
- 🔄 **In progress** — folder set up, Session ≥ 1 in progress
- ⬜ **Pending** — not started
- ⏸ **Paused** — set up but learning suspended (state preserved)

---

## Module folder structure (per global P17)

Each module folder contains **5 root files + 1 lazy sub-directory**:

| File / dir | Role |
|------------|------|
| `reference.md` | Entry point, source map, roadmap, current status |
| `preferences.md` | Module-specific teaching rules (P101+) — inherits from global |
| `mission.md` | Why, success, constraints, out-of-scope (per `~/.agents/skills/teach/MISSION-FORMAT.md`) |
| `audit_flags.md` | **Source-code** bugs and design flaws found while learning (append-only) |
| `session_log.md` | Per-session progress record (append-only) |
| `learning-records/` | **Learner** insights (`0001-slug.md`, …) — created lazily, per `~/.agents/skills/teach/LEARNING-RECORD-FORMAT.md` |

`audit_flags.md` and `learning-records/` are distinct: audit flags catch
**what's wrong in the code**; learning records capture **what the learner
has internalized**. Don't mix them.

The `teach` skill itself lives at `~/.agents/skills/teach/` (the
canonical opencode skills location) — not inside `learning_sentinel/`.

---

## Modules

| # | Folder | Source path in `~/projects/sentinel/` | Status | Mission | Progress | Module prefs |
|---|--------|----------------------------------------|--------|---------|----------|--------------|
| 01 | `01_data_module_v2/` | `data_module/` | 🔄 | [→](modules/01_data_module_v2/mission.md) | **6/36** completed (Sessions 01–06 done, Session 07 in progress) | [P101–P110](modules/01_data_module_v2/preferences.md) |
| 02 | `02_preprocessing_ml/` | `ml/src/preprocessing/` | ⬜ | — | — | — |
| 03 | `03_data_extraction_ml/` | `ml/src/data_extraction/` | ⬜ | — | — | — |
| 04 | `04_datasets_ml/` | `ml/src/datasets/` | ⬜ | — | — | — |
| 05 | `05_models_gnn/` | `ml/src/models/gnn_encoder.py` | ⬜ | — | — | — |
| 06 | `06_models_transformer/` | `ml/src/models/transformer_encoder.py` | ⬜ | — | — | — |
| 07 | `07_models_fusion_classifier/` | `ml/src/models/fusion_layer.py` + `sentinel_model.py` | ⬜ | — | — | — |
| 08 | `08_training/` | `ml/src/training/` | ⬜ | — | — | — |
| 09 | `09_inference/` | `ml/src/inference/` | ⬜ | — | — | — |
| 10 | `10_contracts_solidity/` | `contracts/` | ⬜ | — | — | — |

> **Naming note:** the ML-side modules use `_ml` suffix where the directory
> name collides with a data_module subpackage (e.g. `preprocessing/` exists
> in both `data_module/sentinel_data/` and `ml/src/`). The data_module
> `preprocessing/` is Stage 1b of the v2 pipeline; the ML `preprocessing/`
> is `graph_extractor.py` + `graph_schema.py`. Different code, different
> module folders.

---

## Why this order

The order is **pipeline order for the data layer, then model layer, then
contracts**:

1. **01_data_module_v2** first because it's the live work (Stage 7 in
   progress, Run 11 launch 2026-08-18) and the v2 export seam is the
   critical-path for Run 10/11.
2. **02–04 (ML preprocessing/extraction/datasets)** next because the
   model-layer modules (#05–09) depend on understanding what the
   preprocessing produces.
3. **05–07 (models: GNN, transformer, fusion+classifier)** in
   data-flow order: GNN produces node embeddings → transformer produces
   token embeddings → fusion combines them → 4-eye classifier produces
   predictions.
4. **08 (training)** because trainer.py consumes the model architecture.
5. **09 (inference)** because predictor.py is the production path.
6. **10 (Solidity contracts)** last because it's the *target* being
   audited — understanding what the model is detecting is more useful
   after understanding the model itself.

---

## Cross-references

- **Top-level entry:** [`README.md`](./README.md)
- **Global teaching rules:** [`preferences.md`](./preferences.md) (P1–P20)
- **Read-only source project:** `~/projects/sentinel/`
- **Live project memory:** `~/.claude/projects/-home-motafeq-projects-sentinel/memory/MEMORY.md`
- **v2 proposal docs:** `~/projects/sentinel/docs/proposal/Data_Module_Proposals/`
- **Source-of-truth for the data_module:** [`modules/01_data_module_v2/reference.md`](modules/01_data_module_v2/reference.md)
- **`teach` skill (canonical location):** `~/.agents/skills/teach/`
