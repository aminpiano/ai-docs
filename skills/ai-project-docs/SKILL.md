---
name: ai-project-docs
description: |
  Generate or update AI-optimized project documentation using agent teams.
  Creates structured, token-efficient docs that enable ANY AI to immediately work on the project.

  Two modes:
  - Full Generation: scout → write team → review team (new projects or major changes)
  - Update Mode: audit team → update write team → update review team (incremental changes)

  Uses Claude Code's agent team feature.
  Main session acts as lightweight orchestrator — all heavy work delegated to agents via files.

  Use when: (1) Setting up a new project for AI collaboration, (2) User asks to "document this project for AI", (3) Creating architecture docs, (4) Making project AI-readable, (5) "Create AI docs", "generate project map", "make this AI-friendly", (6) "Update AI docs", "sync docs with code", (7) "Check which docs need updating"
---

# AI Project Documentation Generator (Team Edition)

Generate or update 12 AI-optimized documentation files in `ai-docs/` using agent teams.

**Architecture**: Main session is a lightweight orchestrator. All heavy lifting happens in spawned agents. Information flows through files, not the main session's context.

## Documentation Principles

- **Language**: Always write in English. These are reference docs for AI agents — no localization needed regardless of the project's human-facing language.
- **Format**: Structured, machine-parseable formats only. No prose paragraphs. Priority: table > bullet list > code block > prose.
- **No hard line limits**: Write as much as needed for completeness. If a single doc grows unwieldy, split into sub-files with an INDEX.

## Two Modes

### Full Generation Mode

For new projects or major restructuring. Generates all 12 documents from scratch.

```
Main Session (Orchestrator)
  │
  ├─ Step 0: Preflight + Mode Selection
  │
  ├─ Step 1: Scout Team (ai-docs-scout)
  │    ├─ scout-1 ──→ explore business logic (07)
  │    ├─ scout-2 ──→ explore data + API (05, 06)
  │    ├─ scout-3 ──→ explore infrastructure (01-04, 11)
  │    ├─ scout-4 ──→ explore quality + ops (08-10)
  │    │   (all scouts done)
  │    ├─ synthesizer ──→ merge reports → ai-docs/.skeleton.md
  │    └─ All done → shutdown team
  │
  ├─ Step 2: Write Team (ai-docs-gen)
  │    ├─ doc-writer-1 ──→ 01, 02, 03, 04
  │    ├─ doc-writer-2 ──→ 05, 06, 07
  │    ├─ doc-writer-3 ──→ 08, 09, 10, 11
  │    └─ All done → shutdown team
  │
  ├─ Step 3: Review Team (ai-docs-review)
  │    ├─ reviewer-1 ──→ deep review + fix 01, 02, 03, 04
  │    ├─ reviewer-2 ──→ deep review + fix 05, 06, 07
  │    ├─ reviewer-3 ──→ deep review + fix 08, 09, 10, 11
  │    │   (all reviewers done)
  │    ├─ cross-checker ──→ §/cross-ref/shared-facts verify + 00_INDEX.md
  │    └─ All done → shutdown team
  │
  └─ Step 4: Report to user
```

### Update Mode

For incremental changes. Audits existing docs, identifies what needs updating (including doc-internal issues), and selectively edits affected documents only.

```
Main Session (Orchestrator)
  │
  ├─ Step 0: Preflight + Mode Selection
  │    ├─ Extract previous commit from metadata
  │    ├─ git diff → changed files list
  │    └─ User confirms "Update" mode
  │
  ├─ Step U-1: Audit Team (ai-docs-audit)
  │    ├─ auditor-1 ──→ audit 07 (business logic, solo — heaviest)
  │    ├─ auditor-2 ──→ audit 05-06 (data + API)
  │    ├─ auditor-3 ──→ audit 01-04, 11 (infra + structure + todo)
  │    ├─ auditor-4 ──→ audit 08-10 (quality + operations)
  │    │   (all auditors done)
  │    ├─ synthesizer ──→ merge reports → skeleton update + manifest
  │    └─ All done → shutdown team
  │    → Orchestrator presents Refactoring List to user → confirm
  │
  ├─ Step U-2: Update Write Team (ai-docs-update)
  │    ├─ update-writer-1 ──→ Edit affected docs (partition 1)
  │    ├─ update-writer-2 ──→ Edit affected docs (partition 2)
  │    ├─ (1-3 writers, scaled by affected doc count)
  │    └─ All done → shutdown team
  │
  ├─ Step U-3: Update Review Team (ai-docs-update-review)
  │    ├─ update-reviewer ──→ deep review modified docs
  │    │   (reviewer done)
  │    ├─ cross-checker ──→ cross-ref consistency + 00_INDEX.md update
  │    └─ All done → shutdown team
  │
  └─ Step U-4: Cleanup + Report
```

---

## Tools Used

All tools below are **Claude Code built-in tools** — no custom API. Just call them directly.

| Tool | Purpose |
|------|---------|
| `TeamCreate` | Create a team (= shared task list) |
| `TaskCreate` / `TaskUpdate` / `TaskList` | Manage tasks within the team |
| `Agent` | Spawn a teammate (`subagent_type`, `team_name`, `name`, `prompt`, `description`) |
| `SendMessage` | DM or shutdown request to a teammate (by `name`) |
| `TeamDelete` | Clean up team after all agents shut down |

---

## Step 0: Preflight Checks + Mode Selection

1. **SPEC exists**: Check if `ai-docs/SPEC.md` exists.
   - If exists: proceed.
   - If not exists: Read `references/default-spec.md` and generate SPEC per its instructions, then proceed.
2. **Directory setup**: Ensure `ai-docs/` directory exists (`mkdir -p ai-docs`).
3. **Mode selection**: Check if existing documentation is present.
   - If `ai-docs/00_INDEX.md` exists AND contains a metadata line (`> Generated: ... | Based on commit: <hash>`):
     - Ask user: **"Full regeneration or Update?"**
     - **Full**: proceed to Step 1 (overwrites everything)
     - **Update**: proceed to Update Mode (Step U-1)
   - If no existing docs: Full Generation automatically.
4. **Full Generation preflight** (only when Full mode selected):
   - **Stale state check**: If `ai-docs/.skeleton.md` already exists, warn the user — this may be from a previous incomplete run. Ask whether to continue from existing state or start fresh (delete and regenerate).
   - **Existing docs backup**: If any `ai-docs/0*.md` or `ai-docs/1*.md` files exist, inform the user that full regeneration will overwrite them.
5. **Update Mode preflight** (only when Update mode selected):
   - Extract previous commit hash from `ai-docs/00_INDEX.md` metadata line.
   - Run `git diff --name-only <prev_commit>..HEAD` to get changed files list.
   - Run `git diff --stat <prev_commit>..HEAD` to get change summary.
   - Read `ai-docs/.skeleton.md` section `## E. File-to-Document Map` for initial impact mapping.
   - Present to user: changed file count, initial affected document estimate.
   - Store `PREV_COMMIT`, `CURRENT_COMMIT`, `CHANGED_FILES` for Audit Team.

This is the only step where the main session reads files directly.

---

## Mode Routing

After Step 0 determines the mode:
- **Full Generation** → Read `references/full-mode.md` and follow its instructions (Step 1~4)
- **Update Mode** → Read `references/update-mode.md` and follow its instructions (Step U-1~U-4)
- **SPEC missing** → Read `references/default-spec.md` to generate SPEC first, then re-run Step 0

## Reference Files
- `references/full-mode.md` — Full Generation pipeline (scout → write team → review team)
- `references/update-mode.md` — Update Mode pipeline (audit team → write team → review team)
- `references/default-spec.md` — SPEC auto-generation guide (when SPEC.md is absent)
