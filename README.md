# Sentinel Learning Project

> **A structured deep-dive learning workspace for the SENTINEL smart-contract
> security oracle project.** Read this first, then [`00_INDEX.md`](./00_INDEX.md),
> then [`preferences.md`](./preferences.md), then pick a module.

---

## What this is

`~/projects/learning_sentinel/` is a **persistent learning workspace** that
mirrors the structure of the SENTINEL project so each piece of source code
can be learned in isolation, with full context preserved across sessions.

| What | Where | Rule |
|------|-------|------|
| **Read-only** source code | `~/projects/sentinel/` | We never modify it. We only read it. |
| **Writable** learning workspace | `~/projects/learning_sentinel/` | This is where we write. |
| **Live project memory** | `~/.claude/projects/-home-motafeq-projects-sentinel/memory/` | Ali's session state. Not edited by Claude. |

The SENTINEL project itself is **read-only from this workspace's perspective**.
Any time you want to look at a file, you read it from `~/projects/sentinel/`.
Any time you want to write something, you write it in `~/projects/learning_sentinel/`.

---

## The Read-Only / Write-Here Boundary

This boundary is a **hard rule** (see global [`P19`](./preferences.md#p19--read-only-source-write-only-here-hard-rule)).
Be explicit about which side of the boundary every action is on.

### ✅ Read-only actions (allowed in `~/projects/sentinel/`)

- Open and read files (`Read`, `cat`, `less`)
- Search content (`grep`, `rg`)
- List files and directories (`ls`, `find`, `Glob`)
- Run read-only tools (`wc`, `git log`, `git show`, `git diff`)
- Run scripts in **read-only** mode (e.g. `sentinel-data run --dry-run`)

### 🚫 Forbidden actions in `~/projects/sentinel/`

- `Write`, `Edit` on any file
- `rm`, `mv`, `cp` to overwrite, `touch`
- `git commit`, `git push`
- Running a script that writes inside the source tree
- Creating, deleting, or renaming files
- Modifying configs (`.env`, `config.yaml`, `pyproject.toml`)

### ✅ All writes go in `~/projects/learning_sentinel/`

- Session notes, learning materials, audit flags
- Code experiments (in `experiments/` or similar)
- Glossary entries, learning records, deep-dive docs
- New module folders

### 🚨 Override

If a task genuinely requires modifying the source — for example, to
verify a bug fix or run a controlled experiment — **stop and ask the
user**. The user can grant a per-action override, but Claude never
assumes permission.

---

## How it's organized

```
~/projects/learning_sentinel/
├── README.md               ← you are here
├── 00_INDEX.md             ← master index of all modules
├── preferences.md          ← global teaching rules (P1–P20)
└── modules/
    ├── 01_data_module_v2/        ← ✅ set up (v2 pipeline: ingest → … → export)
    │   ├── reference.md
    │   ├── preferences.md         (P101–P110 module-specific)
    │   ├── audit_flags.md
    │   └── session_log.md
    ├── 02_preprocessing_ml/      ← ⬜ pending
    ├── 03_data_extraction_ml/    ← ⬜ pending
    ├── 04_datasets_ml/           ← ⬜ pending
    ├── 05_models_gnn/            ← ⬜ pending
    ├── 06_models_transformer/    ← ⬜ pending
    ├── 07_models_fusion_classifier/  ← ⬜ pending
    ├── 08_training/              ← ⬜ pending
    ├── 09_inference/             ← ⬜ pending
    └── 10_contracts_solidity/    ← ⬜ pending
```

Each module folder follows the **4-file pattern** (see [P17](./preferences.md#p17--module-folder-spec-4-file-pattern)):

| File | Role |
|------|------|
| `reference.md` | Entry point. What this module is, where it lives, file inventory, current status, roadmap, critical concepts, deferred items, cross-references. |
| `preferences.md` | Module-specific teaching rules (P101+). Inherits from global. Most modules need 0–3 module-specific rules. |
| `audit_flags.md` | Module-local `[AUDIT]` flag log. Append-only. Every bug/design issue found while learning goes here. |
| `session_log.md` | Per-session progress record. Append-only. What was taught, recall results, gaps closed. |

---

## How to start

1. **Read this file** (you are here).
2. **Read [`00_INDEX.md`](./00_INDEX.md)** to see all modules + their status.
3. **Read [`preferences.md`](./preferences.md)** to understand the global teaching rules (P1–P20).
4. **Pick a module.** Currently only `01_data_module_v2/` is set up.
5. **Read that module's `reference.md`** for the big picture + roadmap.
6. **Read that module's `preferences.md`** for module-specific rules.
7. **Start Session 1** from the roadmap. After each chunk, update
   `audit_flags.md` (if any flags raised) and `session_log.md` (always).

To **resume** a module after a break: read all 5 files in the module folder
first. The current state of the journey lives in the spec files, not in
conversation memory.

To **set up a new module** (e.g. `02_preprocessing_ml/`):
1. Create the folder.
2. Copy the 4-file pattern from `01_data_module_v2/` as a template.
3. Replace the content with facts from the new module's source code
   (read the source first — do not assume).
4. Update [`00_INDEX.md`](./00_INDEX.md) to reflect the new module.
5. Add module-specific rules to its `preferences.md` (P101+).

---

## The Spec File Update Protocol

This protocol defines exactly when, how, and what to update in the 4 spec
files (3 files at the module level + 1 global `preferences.md`). It is
followed on every session, every response that triggers an update.

### When to update

| Trigger | File(s) to update | Timing |
|---------|-------------------|--------|
| User states a new preference (global) | `preferences.md` (global) | **Immediately** — before continuing teaching |
| Claude observes a new global teaching pattern | `preferences.md` (global) | End of the current response |
| User states a module-specific preference | module's `preferences.md` | **Immediately** |
| An `[AUDIT]` flag is raised inline | module's `audit_flags.md` | **Immediately** — same response that raised it |
| A chunk finishes (teaching delivered, questions posted) | module's `session_log.md` | End of that response |
| Current status changes (chunk complete, phase done) | module's `reference.md` (Current Status) | End of that response |
| A global preference is refined or clarified | `preferences.md` (global) | Immediately, with a note on what changed |
| Source code changes (file added, function moved) | module's `reference.md` (source map) | Same session |
| A new module is set up | `00_INDEX.md` | Same session |

### How to update

- **Global `preferences.md`** — Append new `### P#` section at the bottom.
  Never rewrite existing ones silently; add a "(refined: ...)" note if
  clarifying.
- **Module `preferences.md`** — Append new `### P10#` section at the
  bottom. Same rules.
- **Module `audit_flags.md`** — Append new `## A#` entry at the bottom.
  Full format: File, Location, Issue, Fix, Severity, Status, Raised.
  Never delete or edit past entries.
- **Module `session_log.md`** — Append new `## Session N` block. Include:
  file/lines, concepts taught, warm-up results, gaps closed, audit flags
  raised.
- **Module `reference.md`** — Update the `Current Status` section inline.
  Update roadmap chunk markers (✅ → 🔄 → ⬜).
- **`00_INDEX.md`** — Update the module status table inline.

### What must be in each entry

**Preferences entry minimum:**
```
### P# — Short Title
What the rule is.
When it applies.
Format/example if relevant.
```

**Audit flag entry minimum:**
```
## A# — File — Short description
**File:** path
**Location:** function/line
**Issue:** what is wrong and why it matters
**Fix:** concrete fix
**Severity:** Low / Medium / High
**Status:** Open / Noted / Fixed
**Raised:** Session N, Chunk N
```

**Session log entry minimum:**
```
## Session N — Chunk name
**File:** path (lines)
**Concepts taught:** bullet list
**Warm-up recall:** pass/fail per question, gaps noted
**Challenge questions:** answered Y/N, gaps closed
**Audit flags raised:** A# list
```

### Rules for Claude

1. **At the start of every teaching session:** read all 5 files in the
   active module's folder to restore full context. No assumptions — the
   state of the journey lives in the spec files, not in conversation
   memory.
2. **Preferences are non-negotiable constraints.** Every teaching
   response must comply with ALL active preferences in the relevant
   `preferences.md` (global + module). Check before writing.
3. **Audit flags must be raised inline** during teaching using the format:
   > **[AUDIT] A#** — description
   Then immediately append to the module's `audit_flags.md`. Never delay.
4. **Session log updates** happen after each chunk is fully delivered and
   questions posted — not mid-chunk.
5. **Never delete entries** from `audit_flags.md` or `session_log.md`.
   Only append.
6. **Source code is the only source of truth.** Do not invent or assume
   facts. If you don't know, read the file.

### Rules for the User

- State new preferences at any time — Claude will add them to the right
  `preferences.md` immediately.
- Challenge question answers trigger gap-fill teaching (P2) and a
  session log update.
- To resume a session after a break: say "resume" + the module name.
   Claude reads all 5 files in that module first.
- To set up a new module: name it + the source path, then Claude
   explores the source and creates the 5 files.

---

## Current status (2026-06-13)

- **Active module:** `01_data_module_v2/` (set up, not yet started)
- **Global prefs:** P1–P20 active (P17 added 2026-06-13 — module folder spec; P18–P20 added same day)
- **Pending modules:** 9 (all ⬜ — see [`00_INDEX.md`](./00_INDEX.md))
- **Audit flags raised (this project):** 0
- **Sessions logged (this project):** 0

---

## See also

- [`00_INDEX.md`](./00_INDEX.md) — master module index
- [`preferences.md`](./preferences.md) — global teaching rules
- [`modules/01_data_module_v2/`](./modules/01_data_module_v2/) — first module
- **Read-only source:** `~/projects/sentinel/`
- **Live project memory:** `~/.claude/projects/-home-motafeq-projects-sentinel/memory/MEMORY.md`
