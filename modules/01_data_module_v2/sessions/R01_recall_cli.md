# Recall R1 — CLI Architecture & Orchestrator (Sessions 01–04)

**Purpose:** Recall, review, relearn — consolidate 4 sessions into one
**Estimated review time:** 20 min
**Difficulty:** Core (must know for interviews)

---

## 1. The Big Picture

SENTINEL is a CLI-first system. The `sentinel-data` command routes to stages via `argparse` subparsers. Each stage (ingest → preprocess → represent → label → verify → split) is a Python function called by a CLI handler in `cli.py`. Config is loaded from `config.yaml`, sources are iterated via `_enabled_sources()`, and every stage has a dry-run mode.

---

## 2. Architecture Diagram

```
CLI (cli.py)
  │
  ├── sentinel-data ingest
  │     └── _run_ingest() → ingest_source() / ingest_all()
  │           └── connectors.BaseConnector.pull() + manifest write
  │
  ├── sentinel-data preprocess
  │     └── _run_preprocess() → preprocess_source() / preprocess_all()
  │           └── PreprocessingPipeline.run() (5 steps)
  │
  ├── sentinel-data represent
  │     └── _run_represent() → represent_source()
  │           └── graph_extractor + tokenizer
  │
  ├── sentinel-data label
  ├── sentinel-data verify
  └── sentinel-data split
```

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| `argparse` subparser tree | 02 | ☐ |
| `_run_*` handler pattern | 03 | ☐ |
| `_enabled_sources()` filter | 04 | ☐ |
| Config precedence: CLI > env > config.yaml > defaults | 02 | ☐ |
| `dry_run` mode — prints without writing | 04 | ☐ |
| `--sample N` — limit to first N files | 04 | ☐ |
| STAGE_DESCRIPTIONS dict | 03 | ☐ |
| Re-export `__all__` from `__init__.py` | 01 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: Why does every `_run_*` handler load `cfg` from `config.yaml` instead of receiving it as a parameter?**

The CLI entry point reads config and data_dir ONCE at the top of `main()`, then passes both to each handler. This means config loading is centralized — if you change how config is loaded, all handlers benefit without modification.

**Q2: What happens if you run `sentinel-data preprocess --source X` without running `sentinel-data ingest --source X` first?**

`preprocess_source()` checks `raw_dir / "ingestion_manifest.json"` exists. If not, raises `FileNotFoundError` with message "Run `sentinel-data ingest --source X` first." This is the hard gate between stages.

**Q3: The `_enabled_sources()` function filters `cfg["sources"]` by `enabled: true`. Where is this used, and what pattern does it establish?**

Used by every stage that iterates sources (preprocess_all, represent_all, label_all, etc.). It establishes the "config-driven selection" pattern: enable/disable sources in YAML, no code changes.

**Q4: What's the difference between `--all` and `--source X` in the CLI?**

`--all` iterates `_enabled_sources()` and runs the selected stage for each. `--source X` runs for exactly one source. If you pass both, `--source X` wins (single source). This is the "specific > all" precedence rule.

---

## 5. Session-by-Session Memory Triggers

- **S01:** `__init__.py` re-exports, `STAGES` ordered list, stage-version info
- **S02:** `argparse.ArgumentParser` tree, `config.yaml` loading, `Path` resolution
- **S03:** `_run_*` handlers, `STAGE_DESCRIPTIONS`, flag parsing per stage
- **S04:** `_enabled_sources()`, `dry_run` guard, `--sample N` implementation

---

## 6. What To Remember

1. **CLI is the entry point.** Every pipeline operation starts from `cli.py`.
2. **Config is loaded once** at the top and threaded through all handlers.
3. **Stages are sequential gates.** ingest → preprocess → represent → label → verify → split. Each stage checks the previous stage's output exists.
4. **`_enabled_sources()` is the source filter** used by every multi-source stage.
5. **Dry-run is universal.** Every stage checks `dry_run` before writing.
