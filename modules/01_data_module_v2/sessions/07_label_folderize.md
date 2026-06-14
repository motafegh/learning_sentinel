# Session 07 — Label Folderize (`ingestion/label_folderize.py`)

**Date:** 2026-06-14
**Session number:** 07 of 36
**Mode:** Understand (standalone utility — no CLI entry point; called from connectors or scripts)
**Estimated study time:** 45 min teaching + 15 min Q&A
**Status:** 🟡 In progress — questions pending

---

## §0 Recap of Session 06 + Model Answers to Q1–Q4

### §0.1 Recap of Session 06

Session 06 covered the connectors strategy pattern. Key concepts: **(1)** `BaseConnector` ABC with template method `pull()` wrapping `_pull()` + timing. **(2)** `find_sol_files` shared static method — `include_subdirs` limits search space, `exclude_subdirs` filters results. **(3)** Git connector: full clone (pinned) vs shallow clone (unpinned); `post_clone_cmd` hook via `cfg.extra`. **(4)** Manual connector: 3 materialization cases (zip → extract + symlink/copy, dir → symlink/copy, glob → resolve). **(5)** `_extract_zip` strips `__MACOSX/` + defends against zip-slip. **(6)** 3/5 connectors are stubs (etherscan, huggingface, zenodo).

### §0.2 Model answer to Q1 [Mechanism]

> **Q:** In `git_connector._pull()`, if `repo_dir` already exists, the connector skips cloning. What happens if the upstream repo has new commits since the last ingestion? Is the data silently stale?

**Model answer:** Yes, the data is silently stale. The existing `repo_dir` is reused without fetching — `_current_commit()` returns the old HEAD SHA. The manifest records this as the resolved pin. No error or warning is raised. The only signal of staleness is the freshness check (`freshness.py:_git_upstream_head`), and only for pinned sources — unpinned sources tracking HEAD have no way to detect that upstream moved. This is a deliberate trade-off: idempotent re-runs skip the expensive clone for speed, at the cost of potentially stale data. The freshness check is the intended mitigation.

### §0.3 Model answer to Q2 [Pattern]

> **Q:** `materialize_staging` treats `mode="symlink"` for zips as "extract + symlink" but `mode="copy"` as "extract in place". Why this asymmetry?

**Model answer:** Zips cannot be "symlinked" — the content must be extracted. The `symlink` mode means "extract to a separate temp dir (`repo_dir__zip_extracted/`) and symlink that as `repo_dir`", preserving the option to delete the symlink without re-extracting the zip. The `copy` mode extracts directly to `repo_dir`. The asymmetry is pragmatic: the mode describes the INTENT (lightweight reference vs full copy), not the mechanism. For zips, both modes extract — they differ in WHERE the data goes and whether the symlink is the user-facing handle.

### §0.4 Model answer to Q3 [Portable🔵]

> **Q:** If `include_subdirs=["buggy_contracts", "results"]` and `exclude_subdirs=["results"]`, how are files in `results/` handled — does include or exclude win?

**Model answer:** Exclude wins. `include_subdirs` determines which dirs are entered (the SEARCH SPACE). Both `buggy_contracts/` and `results/` are entered. Files in `results/` are found by `rglob("*.sol")`. Then the `exclude_subdirs` blocklist filters the output — files whose top-level subdir is `"results"` are skipped. So a dir in both lists gets included (entered) then excluded (filtered). This interaction is NOT documented in code comments — it must be deduced from the control flow. If the same dir is in both lists, the include is redundant because exclude would catch it anyway.

### §0.5 Model answer to Q4 [Mechanism]

> **Q:** The manual connector's `resolved_pin` falls back to `st_mtime`. How does this affect the manifest's reproducibility guarantee? If you re-ingest without changing any `.sol` files but `st_mtime` changes, does the manifest differ?

**Model answer:** Yes, the manifest differs — `resolved_pin` changes even though the `.sol` files are identical. `st_mtime` is the staging path's modification timestamp; touching the staging dir (add/delete a file, even an irrelevant one) changes it. This undermines the reproducibility guarantee: two runs with the same source data produce different manifests. The fix would be to compute a content hash over the staging directory instead of using `st_mtime`. In practice, the impact is low because: (a) `cfg.pin` takes priority over `st_mtime` — most manual sources set pin to a version string like `"v1.0"`; (b) the `.sol` file content hasn't changed, so SHA-256s in the manifest are still valid; only the metadata field `resolved_pin` changes.

### §0.6 Your answers to Q1–Q4 (to be filled in after you respond)

**Your answer to Q1:** _pending_

**Your answer to Q2:** _pending_

**Your answer to Q3:** _pending_

**Your answer to Q4:** _pending_

---

## §1 What This Chunk Covers (per global P5)

| File | LOC | Role |
|------|-----|------|
| `label_folderize.py` | 173 | 1 function + 1 dataclass: creates per-class symlink folders from a labels CSV |

**Why this module exists (P0):** Some DAPP vulnerability datasets (DIVE, SmartBugs Wild, Bastet) distribute contracts and labels in **separate files** — a directory of `.sol` files + a CSV mapping IDs to class labels. The downstream pipeline (crosswalk YAMLs, splitter stratification) expects a **folder structure** where each class has its own directory. `label_folderize.py` bridges this gap by creating per-class symlinks.

**When this runs:** Not as a CLI stage. It's called from the DIVE connector (or similar) via `cfg.extra.get("label_folderize")` after materialization, or invoked manually from a notebook/script. It's a **support utility** within the ingestion subpackage.

---

## §2 The Problem: Labels in a Separate File

```
DIVE layout (flat):
  data/raw/dive/repo/
    1.sol
    2.sol
    ...
    22332.sol
  data/raw/dive/repo/labels.csv
    contractID,Reentrancy,AccessControl,...  ← class labels in a CSV

What the pipeline expects:
  data/raw/dive/repo/__source__/         ← the canonical .sol files
    1.sol
    2.sol
    ...
  data/raw/dive/repo/Reentrancy/         ← per-class symlink folders
    1.sol  → ../../__source__/1.sol
    42.sol → ../../__source__/42.sol
  data/raw/dive/repo/AccessControl/
    7.sol  → ../../__source__/7.sol
    ...
```

The connector (`manual_connector.py`) materializes the data. `label_folderize.py` converts flat → folderized by reading the CSV and creating symlinks.

---

## §3 `FolderizationResult` — The Return Type (lines 48-70)

```python
@dataclass
class FolderizationResult:
    contracts_seen: int = 0
    symlinks_created: int = 0
    classes_present: list[str] = None
    multi_label: int = 0
    unparseable_labels: int = 0
    files_moved: int = 0

    def __post_init__(self):
        if self.classes_present is None:
            self.classes_present = []
```

**6 metrics:**
- `contracts_seen`: labeled contracts processed (not total `.sol` files — only ones with a CSV row)
- `symlinks_created`: total symlinks across all class folders (a multi-label contract creates multiple)
- `classes_present`: sorted list of classes that have at least one contract
- `multi_label`: contracts with >1 positive class
- `unparseable_labels`: CSV rows with empty `id_column` (data quality signal)
- `files_moved`: `.sol` files relocated from `repo_dir` root into `__source__/`

**`__post_init__`** handles the mutable default issue: `list[str] = None` → default to `[]` in post-init. Standard Python pattern to avoid shared mutable defaults.

---

## §4 `folderize_by_labels` — The 2-Step Process (lines 73-173)

```python
# Learning mode: Understand | Step 1: move flat files → Step 2: create symlinks from CSV
def folderize_by_labels(
    repo_dir: Path,
    labels_csv: Path,
    id_column: str,
    class_columns: list[str],
    source_subdir: str = "__source__",
) -> FolderizationResult:
```

### §4.1 Step 1 — Move flat files into `__source__/` (lines 104-130)

```python
# Learning mode: Master | Idempotent — only moves stragglers on re-run
if source_subdir and source_dir.exists() is False:
    # First run: move ALL flat .sol files into source_dir
    flat_files = [p for p in repo_dir.glob("*.sol")]
    if flat_files:
        source_dir.mkdir(parents=True, exist_ok=True)
        for p in flat_files:
            dest = source_dir / p.name
            if not dest.exists():
                shutil.move(str(p), str(dest))
                result.files_moved += 1
elif source_subdir and source_dir.exists():
    # source_dir already exists. Move stragglers (files added since last run).
    for p in repo_dir.glob("*.sol"):
        dest = source_dir / p.name
        if not dest.exists():
            shutil.move(str(p), str(dest))
            result.files_moved += 1
```

**Two paths:**
1. `source_dir` doesn't exist → first run. Move ALL flat `.sol` files from `repo_dir` root into `__source__/`.
2. `source_dir` exists → re-run. Move only stragglers (files that somehow ended up at root since last run).

**Idempotency:** `if not dest.exists()` prevents re-moving files already in `__source__/`. On re-run, only new files (added by re-ingest) get moved.

**Why move instead of copy:** The flat `.sol` files are the SOURCE files that the connector materialized. Moving them preserves the canonical location under `__source__/` without duplicating data. The symlinks in step 2 reference these moved files — there's only one copy on disk.

### §4.2 Step 2 — Create per-class symlinks from CSV (lines 132-172)

```python
# Learning mode: Understand | One CSV row per contract; multi-label creates symlinks in multiple class dirs
with open(labels_csv, newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        contract_id = row.get(id_column, "").strip()
        if not contract_id:
            result.unparseable_labels += 1
            continue
        result.contracts_seen += 1

        # Determine positive classes
        positive_classes = []
        for c in class_columns:
            v = (row.get(c, "") or "").strip()
            if v in ("1", "true", "True", "yes"):
                positive_classes.append(c)
        if len(positive_classes) > 1:
            result.multi_label += 1
        if not positive_classes:
            continue  # no labels — skip (don't create a class folder entry)

        src_path = source_dir / f"{contract_id}.sol"
        if not src_path.exists():
            continue  # skip silently — more IDs in CSV than files

        for cls in positive_classes:
            cls_dir = repo_dir / cls
            cls_dir.mkdir(parents=True, exist_ok=True)
            link_path = cls_dir / src_filename
            if not link_path.exists():
                link_path.symlink_to(Path("..") / source_subdir / src_filename)
                result.symlinks_created += 1
```

**Row processing logic:**
1. Read `contractID` from the CSV row (configurable via `id_column`)
2. Check each `class_columns` value — accept `"1"`, `"true"`, `"True"`, `"yes"` as positive
3. Multi-label counting: if a contract has 2+ positive classes, increment `multi_label`
4. Empty-label contracts are silently skipped (no class folder entry)
5. Missing `.sol` files are silently skipped (CSV may have more rows than files)
6. For each positive class: create `repo_dir/<Class>/<id>.sol` → `../../__source__/<id>.sol`

**Symlink format:** `Path("..") / source_subdir / src_filename`
- For `source_subdir="__source__"`: `../../__source__/42.sol`
- For `source_subdir="buggy_contracts"`: `../../buggy_contracts/42.sol`

The relative path is always 2 levels up from `repo/<Class>/<id>.sol` to `repo/<source_subdir>/<id>.sol`.

### §4.3 Multi-label handling

A contract with 3 positive labels (`Reentrancy`, `AccessControl`, `FlashLoan`) results in:

```
repo/__source__/42.sol                          ← canonical (real) file
repo/Reentrancy/42.sol      → ../../__source__/42.sol
repo/AccessControl/42.sol   → ../../__source__/42.sol
repo/FlashLoan/42.sol       → ../../__source__/42.sol
```

All 3 symlinks point to the same canonical file. Downstream tools can scan a single class folder (e.g., `repo/Reentrancy/`) without seeing contracts from other classes — clean for class-balanced sampling.

---

## §5 Flat vs Folderized Sources

| Source type | Example | `source_subdir` | Layout after folderize |
|------------|---------|-----------------|----------------------|
| Flat (labels in CSV) | DIVE, SmartBugs Wild | `"__source__"` | `.sol` files moved from root → `__source__/`; per-class symlinks created |
| Already folderized | SolidiFI (`buggy_contracts/`) | `"buggy_contracts"` | No files moved (none at root); symlinks point to `../../buggy_contracts/<id>.sol` |

For folderized sources like SolidiFI where the canonical files are already in a subdirectory (`repo/buggy_contracts/`), set `source_subdir="buggy_contracts"`. Step 1 finds no flat files at root (they're already in `buggy_contracts/`), so no move happens. Step 2 creates symlinks pointing to `../../buggy_contracts/<id>.sol`.

**`include_subdirs` interaction:** After folderization, `repo_dir` contains only `__source__/` and per-class subdirs (plus the labels CSV). Setting `include_subdirs=["Reentrancy"]` on a connector would scan only the `Reentrancy/` folder — useful for class-specific preprocessing or balanced sampling.

---

## §6 Idempotency Contract

```
First run:
  repo/
    1.sol  → moved to __source__/1.sol
    2.sol  → moved to __source__/2.sol
    labels.csv
  repo/__source__/
    1.sol (real)
    2.sol (real)
  repo/Reentrancy/
    1.sol → ../../__source__/1.sol
  repo/AccessControl/
    2.sol → ../../__source__/2.sol

Re-run (same data):
  repo/__source__/   ← already exists, no files at root to move
    1.sol (real)
    2.sol (real)
  repo/Reentrancy/   ← symlinks already exist, no new files created
    1.sol → ...
```

**Safe to re-run after re-ingest:** If the connector re-materializes new `.sol` files, Step 1 moves the new files into `__source__/` (they're not there yet). Step 2 creates symlinks for any new contracts. Old symlinks remain unchanged.

**Safe to re-run without re-ingest:** No files at root → no move. Existing symlinks exist → no re-creation.

---

## §7 Full Trace (Connector → Label Folderize)

```
DIVE source in config.yaml:
  connector: manual
  extra:
    staging_path: /net/data/dive/v2.1/solidity_files.zip
    label_folderize:
      labels_csv: /net/data/dive/v2.1/labels.csv
      id_column: contractID
      class_columns: [Reentrancy, AccessControl, ...]

CLI: sentinel-data ingest --source dive

ingest.py:ingest_source("dive", cfg, data_dir)
  → connector = get_connector("manual")
  → connector.pull(source_cfg, raw_dir)
      → ManualConnector._pull(...)
          → materialize_staging(zip_path, repo_dir, extra, ...)
              → _extract_zip(zip, extract_root)  # strip __MACOSX/
              → symlink repo_dir → extract_root
          → find_sol_files(repo_dir, ...)
          → PullResult(sol_files=[...], ...)
  → manifest saved

← label_folderize runs (called from outside, e.g. a post-ingest script)
  folderize_by_labels(
      repo_dir=raw_dir/"dive"/"repo",
      labels_csv=/net/data/dive/v2.1/labels.csv,
      id_column="contractID",
      class_columns=["Reentrancy", "AccessControl", ...],
  )

  Step 1: move flat .sol from repo/ → repo/__source__/
  Step 2: read CSV, create per-class symlinks

Result:
  data/raw/dive/repo/__source__/1.sol, 2.sol, ...
  data/raw/dive/repo/Reentrancy/1.sol → ../../__source__/1.sol
  data/raw/dive/repo/AccessControl/2.sol → ../../__source__/2.sol
  ...
```

---

## §8 3 Things to Lock In (per global P10-C)

1. **`label_folderize` is NOT a CLI stage.** It's a support utility called from the DIVE connector (or a notebook). No `_run_label_folderize` in `cli.py`. It bridges the gap between flat-source datasets (DIVE) and the expected folder structure.

2. **Two-step process: move then symlink.** Step 1 moves flat `.sol` files from `repo_dir` root into `__source__/` (idempotent — only moves stragglers on re-run). Step 2 reads the CSV and creates per-class symlinks. Multi-label contracts get symlinks in multiple class folders, all pointing to the same canonical file.

3. **`source_subdir` adapts to source type.** Flat sources (DIVE) use `"__source__"` (files are moved). Already-folderized sources (SolidiFI with `buggy_contracts/`) use `"buggy_contracts"` (files stay put, symlinks point there). The rest of the pipeline doesn't need to know which layout the source uses — it just sees per-class folders.

---

## §9 Abbreviations introduced this session (per global P12)

- **Flat source** — a dataset where `.sol` files are all in one directory (no folder structure), with labels in a separate CSV
- **Folderized source** — a dataset where `.sol` files are already organized into per-class or per-category subdirectories
- **Idempotent** — running the same operation twice produces the same result as running it once
- **CSV** — Comma-Separated Values (a text file format where each row is a record and columns are separated by commas)
- **Symlink** — symbolic link (a special filesystem entry that points to another file or directory)

---

## §10 Challenge Questions (4 — per global P10-E)

### Q1 [Mechanism]

> In Step 1 (lines 107-124), if `source_subdir=""` (empty string), the `if source_subdir and source_dir.exists()` check is False for both branches. What happens? Are flat files moved? Is the function still usable?

### Q2 [Pattern]

> The function moves flat `.sol` files into `__source__/` (Step 1) and then creates relative symlinks from class folders back to `//../__source__/<id>.sol` (Step 2). Why the two-step approach? Why not create symlinks directly from the flat location without moving?

### Q3 [Portable🔵]

> The CSV row parser accepts `"1"`, `"true"`, `"True"`, `"yes"` as positive class indicators (line 146). But it does NOT accept `1` (integer, without quotes). If the CSV writer outputs numeric `1` instead of string `"1"`, what happens? Is this a potential bug?

### Q4 [Mechanism]

> `label_folderize` is NOT called by any CLI handler or by `ingest_source`. How is it triggered in practice? Trace the path from "user runs `sentinel-data ingest --source dive`" to "per-class symlinks exist."

---

## §11 Connections (what to study next)

- **Session 08 — Preprocessing overview:** `preprocessing/pipeline.py` + `preprocess.py`. The first real subpackage beyond ingestion. You'll see how the pipeline coordinator flattens, compiles, deduplicates, normalizes, segments, and version-buckets the ingested raw contracts.
- **Session 09 — Preprocessing: flattener:** `preprocessing/flattener.py` (311 LOC). The "unconditional chunking" that handles multi-file Solidity projects — this is the most complex preprocessing step.

---

## §12 What "got it" looks like for Session 07

- [ ] Explain why `label_folderize` exists (separate-file datasets like DIVE need per-class folders)
- [ ] Describe the two-step process: what moves in Step 1, what happens in Step 2
- [ ] Explain how `source_subdir` adapts to flat vs folderized sources
- [ ] Trace a multi-label contract through the folderization: how many symlinks, where do they point
- [ ] Explain the idempotency contract: what changes on re-run vs what stays the same
- [ ] Describe the `FolderizationResult` metrics and what each means
- [ ] Explain why `label_folderize` is not a CLI stage (answered Q4)

---

**Next:** answer Q1–Q4 in chat. I'll gap-fill (P2), update the **Your answers** sections, and update `session_log.md` for this session. After Q&A, we move to Session 08 (preprocessing overview).
