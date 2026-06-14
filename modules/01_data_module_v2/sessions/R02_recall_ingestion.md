# Recall R2 — Ingestion (Sessions 05–07)

**Purpose:** Recall, review, relearn — consolidate 3 sessions into one
**Estimated review time:** 20 min
**Difficulty:** Core (must know for interviews)

---

## 1. The Big Picture

Ingestion pulls raw `.sol` files into `data/raw/<source>/`. It uses a strategy pattern (`BaseConnector` ABC) with 5 concrete connectors: Git, Manual, and 3 stubs. After files land on disk, a manifest (`ingestion_manifest.json`) records the file list. Optional label folderization creates per-class symlinks for DIVE-like datasets.

---

## 2. Architecture Diagram

```
config.yaml
  sources:
    dive:      { type: git, ... }
    solidifi:  { type: manual, ... }

      │
      ▼
ingest_source(name, cfg, data_dir)
      │
      ├── connector = _build_connector(cfg["sources"][name])
      │
      ├── connector.pull(raw_dir)               ← template method
      │     ├── _pull()  (clone / extract / copy)
      │     └── timing wrapper
      │
      ├── find_sol_files(raw_dir, include_subdirs, exclude_subdirs)
      │
      ├── write_ingestion_manifest(raw_dir, files, connector, timing)
      │
      └── (optional) folderize_by_labels(labels_csv, ...)  ← Session 07
```

**Connectors:**
- **Git:** `git clone` (pinned=full, unpinned=shallow), `post_clone_cmd` hook
- **Manual:** zip → extract + symlink, dir → symlink/copy, glob → resolve
- **Etherscan, HuggingFace, Zenodo:** stubs (not yet implemented)

---

## 3. Key Concepts Checklist

| Concept | Session | Do you know it? |
|---|---|---|
| `BaseConnector` ABC with `pull()` template method | 05 | ☐ |
| Git connector: pinned full clone vs unpinned shallow | 06 | ☐ |
| Manual: 3 materialization cases (zip, dir, glob) | 06 | ☐ |
| `post_clone_cmd` hook via `cfg.extra` | 06 | ☐ |
| Zip-slip defense: `os.path.commonprefix` check | 06 | ☐ |
| `find_sol_files()` — `include_subdirs` / `exclude_subdirs` | 05 | ☐ |
| `ingestion_manifest.json` — the gate file | 05 | ☐ |
| `folderize_by_labels()` — builds per-class symlinks | 07 | ☐ |
| `__source__/` convention — canonical flat location | 07 | ☐ |
| `_maybe_folderize()` — the bridge from ingest to preprocess | 07 | ☐ |
| Find sol files finds symlinks, not the source files | 07 | ☐ |

---

## 4. Interview-Style Q&A

**Q1: Why is ingestion a 2-step process (pull files THEN scan for .sol files) instead of just pulling and globbing in one pass?**

Separation of concerns. `_pull()` materializes the source (git clone, zip extract, dir copy). `find_sol_files()` is a PURE function that scans the materialized files with `include_subdirs` / `exclude_subdirs` filtering. Separating them means you can re-scan without re-pulling, and you can override the scan scope independently of the pull mechanism.

**Q2: What does `source_subdir=""` do in `folderize_by_labels`, and why would you use it?**

If `source_subdir=""` (empty), files are NOT moved to `__source__/`. They stay at the `repo_dir` root. The symlinks point directly to them via `../../<id>.sol`. This is useful when the repo is already set up with flat files at root and you don't want to reorganize.

**Q3: How does the Git connector handle source update (re-ingest)?**

Unpinned (shallow): does `git fetch --depth 1 && git reset --hard origin/main`. Pinned (full): does `git fetch && git checkout <commit_hash>`. Both rebuild the `repo/` directory in place. The manifest is overwritten with the new file list. Any pre-existing symlinks from folderization are broken (the old `__source__/` files may have different content) — re-running `preprocess` re-folderizes.

**Q4: The 38.8% dedup stat from BCCC — where is that used in ingestion?**

It's not used in ingestion — it's used in the deduplicator's docstring (Session 09). Ingestion doesn't deduplicate. It just pulls files. Dedup happens in Step 3 of preprocessing. The stat justifies why the dedup exists.

---

## 5. Session-by-Session Memory Triggers

- **S05:** `BaseConnector.pull()`, `find_sol_files()`, manifest structure, connector registry dict
- **S06:** Git clone modes, manual cases (zip/dir/glob), `_extract_zip` defense, `post_clone_cmd`
- **S07:** `folderize_by_labels()`, `__source__/` convention, multi-label handling, symlink dedup, `_maybe_folderize` bridge

---

## 6. What To Remember

1. **Strategy pattern** via BaseConnector ABC — each connector is a class with `_pull()`.
2. **Manifest is the gate.** Downstream stages check `ingestion_manifest.json` exists.
3. **Folderization is optional.** Only sources with `labels_csv` in config get it.
4. **`__source__/` convention** = canonical flat file location. Symlinks are views.
5. **3/5 connectors are stubs** — Etherscan, HuggingFace, Zenodo.
6. **Zip-slip defense** checks extracted path is within the extraction dir.
