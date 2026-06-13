# Sentinel Learning Project — Global Header (v2 era)

> **Last revised:** 2026-06-13
> **Project root:** `~/projects/learning_sentinel/`
> **Read-only source project:** `~/projects/sentinel/`
> **Current era:** v2 Data Module build (Stages 0–6 ✅, Stage 7 STUB, Run 11 launch 2026-08-18)
> **Live memory:** `~/.claude/projects/-home-motafeq-projects-sentinel/memory/MEMORY.md`
> **v2 proposal:** `~/projects/sentinel/docs/proposal/Data_Module_Proposals/`
>
> This `preferences.md` is the **global source of truth** for teaching rules (P1–P20).
> Every module folder under `modules/` has its own `preferences.md` that
> **inherits** from this file and may ADD module-specific rules (P101+).
> Global rules always win on conflict.

---

# Preferences — Learning With Claude

All active teaching preferences. Every response must comply with ALL of these.
Preferences are added immediately when stated or observed — never batched.

---

## Learning Mode Framework

### The 4 Modes

| Mode | What you must own | Syntax recall? |
|------|------------------|----------------|
| **Awareness only** | Know it exists, roughly what it does. | Never |
| **Understand the pattern** | WHY it exists + WHAT it produces. High-level how. | No |
| **Master the mechanism** | WHY + WHAT + full step-by-step HOW. Trace data shapes. Answer "what breaks if X?" | No — mechanism, not syntax |
| **Master the mechanism** 🔵 | Same + transcends Sentinel. Permanent ML career toolkit. Explain without Sentinel context. | No — but own it deeply enough to teach it |

No syntax recall ever. Key constants/invariants (e.g. `_GNN_IN_DIM=27`, `add_self_loops=False`) go in "3 Things to Lock In" (P10-C) and are remembered as facts.

### 🔵 Portable Flag
Marks concepts that appear across all ML codebases: residual connections, `.detach()`, `register_buffer`, softmax entropy, multi-head shape invariants, LayerNorm, oversmoothing, etc.
🔵 concepts get a fast-recall reteach every time they reappear in a new context (P11).
Challenge questions for 🔵 blocks test explaining without Sentinel context.

### How Modes Appear in Code

```python
# Learning mode: Master the mechanism | 🔵 Portable

def __init__(self, channels: int, num_phases: int = 3) -> None:
    super().__init__()
    self.attn = nn.Linear(channels, 1, bias=False)          # ← MASTER🔵: 256→1 scorer; no bias (cancels in softmax)
    self.register_buffer("last_weights", torch.zeros(...))  # ← MASTER🔵: buffer contract — survives .to(device)/save/load; NOT a param
    self.last_node_weights: "torch.Tensor | None" = None    # ← UNDERSTAND: plain attr — N varies per batch, can't be a buffer
```

**Annotation key:**
- `# ← MASTER:` — own this mechanism; tested in challenge questions
- `# ← MASTER🔵:` — own as portable career knowledge; tested with "explain without Sentinel context"
- `# ← UNDERSTAND:` — know what it does and why; high-level is enough
- No annotation — awareness only; read and move on

Do not over-annotate. Only mark what genuinely matters.

---

## Preferences

### P1 — Teach + Audit Simultaneously
Actively audit while teaching. If something looks wrong — a bug, design flaw, misleading comment, missed edge case — flag it immediately inline. Never blindly accept existing code.
> **[AUDIT] A#** — concern, why it matters, better approach

---

### P2 — Gap-Fill After Challenge Questions
After answers, identify gaps explicitly. Teach back the missing concept with enough depth to close the gap — never just "correct/incorrect."

---

### P3 — Chunk Large/Complex/Important Files
Complex, large, or important files get chunked by logical units (not line counts). Post challenge questions after each chunk before moving on.
Unconditional: `gnn_encoder.py`, `sentinel_model.py`, `transformer_encoder.py`, and any qualifying file.

---

### P4 — Data Flow Diagrams (where useful)
Include ASCII diagrams when a chunk has multiple steps/transformations hard to follow in prose. Not required for simple single-responsibility helpers.

---

### P5 — Big Picture First for Every New File
Open every new file with: what problem it solves, its role in the system, major sections, input→output data flow. Before any code.

---

### P6 — Cross-File Relationships: Recall or Preview
- **Already taught** → explicitly recall and connect. Don't assume it's remembered.
- **Not yet taught** → name it, one sentence on its role, flag for later. Don't go deep.

---

### P7 — Teach Alternative Approaches with High Educational Value
If a meaningfully different alternative exists with high educational value (common in the field, better trade-offs, standard pattern) — teach it alongside and compare.

---

### P8 — Explicitly Highlight Critical Concepts
> ⚠️ **CRITICAL** — [why this must not be skimmed]
Use when a concept underpins everything else, is a common bug source, or is non-obvious.

---

### P9 — Cross-References to Untaught Code
Never reference code from an untaught file without either:
- **Option A (default):** paste the relevant 3–10 lines inline with context.
- **Option B (guided discovery):** give an exact navigation instruction — file, method, line — and have the user go look. Use when finding it IS the exercise.

---

### P10 — Spaced Repetition and Active Recall
Active recall (being tested) consolidates memory; re-reading does not.

**A — Warm-up (every chunk):** Recall questions from the previous chunk before new material. Minimum 3; scale up with how many distinct concepts that chunk covered. One question per major concept is the guide.
**B — Spaced review (every 3–4 chunks):** Questions from older material at increasing intervals. Minimum 2; scale with how much older ground is due for review.
**C — Lock-in summary:** After teaching, before challenge questions: "3 things to lock in." Key constants and invariants live here.
**D — No re-reading:** Questions must require memory retrieval. If the user needs to look it up, reteach — don't re-read.
**E — Challenge questions (every chunk):** Minimum 3; scale up with chunk complexity. A chunk with 6 distinct mechanisms or portable concepts warrants 6 questions. Never cap at 3 if more is justified. Tag every question:
- `[Pattern]` — purpose/effect, 1–2 sentence answer
- `[Mechanism]` — shape tracing, "what breaks if X?"
- `[Portable🔵]` — explain without Sentinel context / where else does this apply?

---

### P11 — Teach Domain Knowledge Inline
Explain ML/PyTorch/GNN/Solidity concepts inline at first occurrence — never assume prior knowledge. This covers GAT/GNN mechanics, attention, LoRA/PEFT, CodeBERT, pooling, reentrancy, and any domain term that appears.

**🔵 Portable concepts reappearing in a new context** — do not just reference. Give a fast-recall reteach:
> 🔵 **Portable recall — [concept]:** [2–4 sentences: what it is, why it matters here, same/different from last time.]

---

### P12 — Expand Abbreviations on First Use
Expand every abbreviation/acronym on its first use in a chunk.
Examples: GAT → Graph Attention Network, CFG → Control Flow Graph, JK → Jumping Knowledge, LoRA → Low-Rank Adaptation, PEFT → Parameter-Efficient Fine-Tuning, IMP → improvement (repo comment prefix).

---

### P13 — Learning Mode Declaration and Inline Annotations
Every code block must have:
1. **Mode declaration above it:** `# Learning mode: Master the mechanism | 🔵 Portable`
2. **Inline annotations on lines that matter** — see the Framework section above for the annotation key and example.

Annotate only what genuinely matters. Over-annotating defeats the purpose.

---

### P14 — Step-by-Step Mechanism Explanation
Depth scales with mode:
- **Master / 🔵** → full step-by-step: every transformation, shape change, design decision
- **Understand** → one clear paragraph: what in, what out, why
- **Awareness** → one sentence

Never assume the user knows Python, PyTorch, or PyG well enough to infer how a pattern works.

---

### P15 — Learning Materials Folder
The user saves teaching chunks to `~/projects/learning_sentinel/modules/<NN_name>/` — specifically `reference.md` (entry point), `session_log.md` (per-session progress), and `learning-records/NNNN-slug.md` (decision-grade insights, created lazily). Claude writes here (not in the SENTINEL source tree). On resume: read the module's 5 spec files (`reference.md`, `preferences.md`, `mission.md`, `audit_flags.md`, `session_log.md`) for context; `session_log.md` is the authoritative progress record.

---

### P16 — Keep Spec Files Concise
Spec files are read at the start of every session. Keep them short enough to scan quickly — substance over prose. When adding a new preference: state the rule clearly in as few lines as possible. No redundant explanations.

---

### P17 — Module Folder Spec (5 Files + 1 Lazy Subdirectory)

Every module under `modules/` MUST contain exactly 5 files at its root
AND 1 lazily-created sub-directory:

**Root files (all required):**
- `reference.md` — entry point, current status, roadmap, source map
- `preferences.md` — inherits from this global file, may add P101+ rules
- `audit_flags.md` — module-local audit log (append-only)
- `session_log.md` — module-local progress tracker (append-only)
- `mission.md` — module mission: why, success, constraints, out-of-scope.
  Follows the format at `~/.agents/skills/teach/MISSION-FORMAT.md` (per the [`teach` skill](~/.agents/skills/teach/SKILL.md)).

**Sub-directory (created lazily — only when the first record is written):**
- `learning-records/` — decision-grade learning records
  (`0001-slug.md`, `0002-slug.md`, …). Follows
  `~/.agents/skills/teach/LEARNING-RECORD-FORMAT.md`. Append-only. Numbered sequentially.
  **Do not** create the empty directory at module setup time.

The 5 root files are the **persistent workflow memory** of the module.
The `learning-records/` directory is the **persistent learning memory**
(distinct from the workflow files — captures insights, not progress).

All files follow the [Spec File Update Protocol](./README.md#the-spec-file-update-protocol)
(defined in the top-level `README.md`).

**Recommended module sub-directories (created lazily):**

- `learning-records/` — decision-grade learner insights (per the
  `teach` skill). Required convention.
- `sessions/` — self-contained study docs, one per teaching session
  (e.g. `01_orientation.md`, `02_cli_entry_point.md`). Each doc
  contains the full teaching content, 3-things-to-lock-in, challenge
  questions with the user's answers, and cross-references to the
  source files. Written during teaching, not after. Recommended
  convention.
- `chunks/` or `stage_N_*.md` — per-chunk deep-dive material. Optional.

A module without the root 5 files is not set up (P17 hard rule).
A module without `learning-records/` is fine until the first record
is written. A module without `sessions/` is fine — but if you want
the study material persisted, create it after the first session.

**Module `preferences.md` rules:**
- Inherit ALL of P1–P20 by reference (do not copy them)
- May ADD module-specific rules as P101, P102, …
- May NOT override a global rule; if conflict, the global rule wins
- Should be short — most modules need 0–3 module-specific rules

**Module `audit_flags.md` and `session_log.md` rules:**
- Append-only (no deletes, no edits to past entries)
- Header explains the format with one example
- Until the first chunk is taught, body = `*(no X yet)*`

**Module `mission.md` rules:**
- Lives at the module root, **not** in `learning-records/`
- Format: `# Mission: {Module Name}` + `## Why` + `## Success looks like` +
  `## Constraints` + `## Out of scope`
- "Why" = 1–3 sentences. The concrete real-world goal.
- "Success looks like" = specific, observable bullets (3–6 typical).
- "Constraints" = the rules that bound how the module is taught.
- "Out of scope" = adjacent topics explicitly NOT pursued.
- Keep it short (a screen or less). If it runs past a screen, it has
  become a plan instead of a compass — split the plan into the roadmap
  in `reference.md` and keep the mission focused.

**Module `learning-records/*.md` rules:**
- Follow `~/.agents/skills/teach/LEARNING-RECORD-FORMAT.md` (numbered, short, decision-grade)
- A learning record = evidence of demonstrated understanding, not a
  journal entry. Coverage ≠ learning. Wait for evidence.
- When a record contradicts an earlier one, mark the old
  `Status: superseded by LR-NNNN` rather than deleting it.
- See P2 (gap-fill) and the audit vs learning-record distinction:
  - `audit_flags.md` captures **bugs and design flaws** in source code
  - `learning-records/*.md` captures **insights the learner has internalized**
  - These are different things; do not put learning insights in
    `audit_flags.md` or bugs in `learning-records/`

The `modules/01_data_module_v2/` folder is the reference implementation
of this pattern.

---

### P18 — Point to the Source Project Root and Sub-Directory

**Rule:** every module's `reference.md` must clearly state, near the top of
the file:

- The **read-only source project root**: `~/projects/sentinel/`
- The **specific sub-directory** being learned in that module
  (e.g. `data_module/`, `ml/src/preprocessing/`, `ml/src/models/`)
- The **one-way relationship** to this learning workspace
  (`~/projects/learning_sentinel/modules/<NN_name>/`)

**Why:** the user must never be confused about which tree is being read
and which tree is being written. The pointer must be in every module,
not just the top-level README, because a session may resume in a
module folder directly.

**Format:** a `## Source` section near the top, or the cross-references
section at the bottom. Either is fine as long as the pointer is
unambiguous.

---

### P19 — Read-Only Source, Write-Only Here (HARD RULE)

**Rule:** the project at `~/projects/sentinel/` is **read-only** from
this learning workspace's perspective.

- ✅ **Allowed** in `~/projects/sentinel/`: read files, run `cat`/`wc`/
  `grep`/`rg`, navigate with `ls`/`find`, search with `rg`/`grep`,
  invoke read-only tools (`pytest -k` on a test, `python -c "import ..."`,
  `git log`/`git show`/`git diff` on commits).
- 🚫 **Forbidden** in `~/projects/sentinel/`: `Write`, `Edit`, `rm`,
  `mv`, `cp` to overwrite, `git commit`, `git push`, running a script
  that writes to disk inside the source tree, modifying configs,
  `touch`ing files, creating new files, deleting files, renaming files.

**All writes** (session notes, learning materials, audit flags, code
experiments, glossary entries, learning records) go in
`~/projects/learning_sentinel/`.

**This is a hard rule, not a suggestion.** If a task appears to require
modifying the source — even temporarily, even in a sub-agent, even
"just to test something" — **stop and ask the user**. The user can
override the rule explicitly for a specific action, but never assume.

**Why:** the source project is the live SENTINEL codebase. A stray
`Edit` could break a run-in-progress or invalidate a checkpoint. The
learning workspace is the safe place to write.

**Companion to:** P17 (module folder spec), P20 (markdown format).

---

### P20 — Learning Material in Markdown, Updated After Each Session

**Rule:** all learning material in this workspace is written in
**`.md` (Markdown)** format. This is the default and the rule for:

- Spec files: `reference.md`, `preferences.md`, `audit_flags.md`, `session_log.md`
- Deep-dive docs: `stage_N_*.md`, `chunk_NN_*.md`
- Glossaries, learning records, notes
- Inline teaching in chat (also Markdown, with code blocks for code)

**Update trigger:** updates happen **after each session or chunk
completes**, per the [Spec File Update Protocol](./README.md#the-spec-file-update-protocol)
in the top-level `README.md`. A chunk is "complete" when the teaching
has been delivered, challenge questions have been posted, and any
audit flags have been raised.

**Exceptions:** interactive lesson files in **HTML** (per the
[`teach` skill](~/.agents/skills/teach/SKILL.md)) require an explicit user decision
to opt in. The default is Markdown; HTML is a deviation that the user
must request.

**Why:** Markdown is plain text, version-controllable, diffable,
readable in any editor, and renders well in `cat`/`less`/GitHub/most
IDEs. Splitting material across many `.html` files would break the
spec-file pattern and make session continuity harder.
