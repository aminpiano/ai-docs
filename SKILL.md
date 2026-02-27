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
  ├─ Step 1: Spawn scout ──→ writes ai-docs/.skeleton.md
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
  │    ├─ auditor-1 ──→ audit docs 01-04 vs actual code
  │    ├─ auditor-2 ──→ audit docs 05-07 vs actual code
  │    ├─ auditor-3 ──→ audit docs 08-11 vs actual code
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

## Tool Reference (Agent Team API)

```json
// Create team
TeamCreate({ "team_name": "ai-docs-gen", "description": "..." })

// Create task
TaskCreate({ "subject": "...", "description": "...", "activeForm": "..." })

// Assign / update task
TaskUpdate({ "taskId": "1", "owner": "agent-name" })
TaskUpdate({ "taskId": "1", "status": "in_progress" })
TaskUpdate({ "taskId": "1", "status": "completed" })

// Set task dependency (cross-checker waits for reviewers)
TaskUpdate({ "taskId": "4", "addBlockedBy": ["1", "2", "3"] })

// Check progress
TaskList()

// Spawn agent
Task({
  "subagent_type": "general-purpose",
  "team_name": "team-name",
  "name": "agent-name",
  "description": "Short description",
  "prompt": "..."
})

// Message agent
SendMessage({ "type": "message", "recipient": "agent-name", "content": "...", "summary": "..." })

// Shutdown agent
SendMessage({ "type": "shutdown_request", "recipient": "agent-name", "content": "All tasks complete" })

// Delete team (after all agents shut down)
TeamDelete()
```

---

## Step 0: Preflight Checks + Mode Selection

1. **SPEC exists**: Check if `ai-docs/SPEC.md` exists.
   - If exists: proceed.
   - If not exists: generate it per §DEFAULT-SPEC (end of this document), then proceed.
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

## Step 1: Scout Agent → Skeleton File

Spawn a single scout agent. Main session does NOT read source code.

```json
Task({
  "subagent_type": "general-purpose",
  "name": "scout",
  "description": "Analyze codebase and generate skeleton",
  "prompt": "<Scout Prompt>"
})
```

### Scout Prompt

```
You are the scout agent for AI project documentation generation.

## Your Mission
1. Read ai-docs/SPEC.md (§0 Hard Rules and §1-2 Phase 0 Skeleton)
2. Explore the entire codebase:
   - Project identity (name, purpose, status)
   - Tech stack (framework, language, DB, auth, styling)
   - Package manifest (package.json / requirements.txt / Cargo.toml / etc.)
   - Directory structure (Glob sweep)
   - Database schema (Prisma, SQL, ORM models, etc.)
   - API surface (routes, server actions, endpoints)
   - Config files (tsconfig, docker-compose, CI/CD, env templates)
   - Entry points (main pages, router structure, middleware)
   - Classification facts: any per-file property that multiple documents might describe independently.
     Grep/read every source file for file-level directives, configuration exports, and module markers.
     Examples by ecosystem (adapt to the actual project):
       * JS/TS: "use client" / "use server", export const dynamic/revalidate/runtime, barrel re-exports
       * Rust: #![no_std], #[cfg(feature)], pub mod boundaries
       * Python: __all__, TYPE_CHECKING blocks, abstract base classes
       * Go: //go:build tags, internal/ package boundaries
     Output a Classification Map table in Shared Facts: file → classification value.
     This is critical — if the skeleton omits a classification, parallel writers will independently guess it and contradict each other.
   - Explicit TODOs/FIXMEs (Grep sweep for TODO, FIXME, HACK, BUG, WORKAROUND)
   - Implicit issues discovered during analysis
3. Generate the skeleton per SPEC §1-2:
   A. Shared Facts (two sub-tables required):
      - Value Facts: versions, paths, ports, counts — scalar values
      - Classification Map: file → category/directive/flag for every file that has a classifiable property
   B. Cross-Reference Map
   C. TODO Registry (pre-assigned IDs)
   D. Section Numbering Scheme (all 11 documents)
   E. File-to-Document Map (optional but recommended for large projects)
      Map key source directories/files to the documents that should reference them.
      This helps writers focus their exploration and reduces redundant codebase scanning.
      Example:
      | Source Path | Primary Document | Secondary |
      |------------|-----------------|-----------|
      | src/routes/ | 06_API | 03_ARCHITECTURE |
      | prisma/schema.prisma | 05_DATA_MODELS | 10_WARNINGS |
      | .env.example | 01_ENVIRONMENT | — |
4. Write to: ai-docs/.skeleton.md
5. Keep it concise — metadata only. The skeleton MUST remain a single file. If approaching the 200-line limit, compress content (shorter descriptions, merge similar items) rather than splitting.

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]

## When Done
Report completion with one-line summary (tech stack, file count, etc.).
```

**After scout completes**:
1. Confirm `ai-docs/.skeleton.md` exists
2. Validate skeleton structure (read ONLY section headers, not body):
   - `## A. Shared Facts` exists (with both `### Value Facts` and `### Classification Map` sub-headers)
   - `## B. Cross-Reference Map` exists
   - `## C. TODO Registry` exists
   - `## D. Section Numbering Scheme` exists
   If any section is missing, message the scout to fix it before proceeding.
3. Do NOT read skeleton body content — keep orchestrator context lightweight.

---

## Step 2: Write Team (ai-docs-gen)

### 2-1. Create Team & Tasks

```json
// Generate unique team name to prevent collision with stale teams from crashed runs
// Format: ai-docs-gen-{YYYYMMDD-HHmmss} (e.g., ai-docs-gen-20260220-143022)
TeamCreate({ "team_name": "ai-docs-gen-{timestamp}", "description": "AI docs parallel writing" })

TaskCreate({ "subject": "Generate 01_ENVIRONMENT.md", "description": "Write ai-docs/01_ENVIRONMENT.md per SPEC and skeleton", "activeForm": "Writing 01_ENVIRONMENT.md" })
TaskCreate({ "subject": "Generate 02_DEPENDENCIES.md", "description": "Write ai-docs/02_DEPENDENCIES.md per SPEC and skeleton", "activeForm": "Writing 02_DEPENDENCIES.md" })
TaskCreate({ "subject": "Generate 03_ARCHITECTURE.md", "description": "Write ai-docs/03_ARCHITECTURE.md per SPEC and skeleton", "activeForm": "Writing 03_ARCHITECTURE.md" })
TaskCreate({ "subject": "Generate 04_STRUCTURE.md", "description": "Write ai-docs/04_STRUCTURE.md per SPEC and skeleton", "activeForm": "Writing 04_STRUCTURE.md" })
TaskCreate({ "subject": "Generate 05_DATA_MODELS.md", "description": "Write ai-docs/05_DATA_MODELS.md per SPEC and skeleton", "activeForm": "Writing 05_DATA_MODELS.md" })
TaskCreate({ "subject": "Generate 06_API.md", "description": "Write ai-docs/06_API.md per SPEC and skeleton", "activeForm": "Writing 06_API.md" })
TaskCreate({ "subject": "Generate 07_BUSINESS_LOGIC.md", "description": "Write ai-docs/07_BUSINESS_LOGIC.md per SPEC and skeleton", "activeForm": "Writing 07_BUSINESS_LOGIC.md" })
TaskCreate({ "subject": "Generate 08_DEBUG.md", "description": "Write ai-docs/08_DEBUG.md per SPEC and skeleton", "activeForm": "Writing 08_DEBUG.md" })
TaskCreate({ "subject": "Generate 09_STANDARDS.md", "description": "Write ai-docs/09_STANDARDS.md per SPEC and skeleton", "activeForm": "Writing 09_STANDARDS.md" })
TaskCreate({ "subject": "Generate 10_WARNINGS.md", "description": "Write ai-docs/10_WARNINGS.md per SPEC and skeleton", "activeForm": "Writing 10_WARNINGS.md" })
TaskCreate({ "subject": "Generate 11_TODO.md", "description": "Write ai-docs/11_TODO.md per SPEC and skeleton", "activeForm": "Writing 11_TODO.md" })
```

### 2-2. Assign Tasks

> **Note**: Task IDs shown below (1-11) assume sequential creation within a fresh team.
> If the Team API assigns different IDs, use the actual IDs returned by TaskCreate.
> The orchestrator should track the mapping: document name → taskId.

```json
// doc-writer-1: Infrastructure (01-04)
TaskUpdate({ "taskId": "1", "owner": "doc-writer-1" })
TaskUpdate({ "taskId": "2", "owner": "doc-writer-1" })
TaskUpdate({ "taskId": "3", "owner": "doc-writer-1" })
TaskUpdate({ "taskId": "4", "owner": "doc-writer-1" })

// doc-writer-2: Data & API (05-07)
TaskUpdate({ "taskId": "5", "owner": "doc-writer-2" })
TaskUpdate({ "taskId": "6", "owner": "doc-writer-2" })
TaskUpdate({ "taskId": "7", "owner": "doc-writer-2" })

// doc-writer-3: Quality & maintenance (08-11)
TaskUpdate({ "taskId": "8", "owner": "doc-writer-3" })
TaskUpdate({ "taskId": "9", "owner": "doc-writer-3" })
TaskUpdate({ "taskId": "10", "owner": "doc-writer-3" })
TaskUpdate({ "taskId": "11", "owner": "doc-writer-3" })
```

### 2-3. Spawn Writers (parallel, single message)

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "doc-writer-1",
  "description": "Write docs 01-04",
  "prompt": "<Writer Prompt — assigned: 01, 02, 03, 04>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "doc-writer-2",
  "description": "Write docs 05-07",
  "prompt": "<Writer Prompt — assigned: 05, 06, 07>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "doc-writer-3",
  "description": "Write docs 08-11",
  "prompt": "<Writer Prompt — assigned: 08, 09, 10, 11>"
})
```

### Writer Prompt Template

```
You are a documentation writer agent in the "ai-docs-gen" team.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (your assigned docs only)
2. ai-docs/.skeleton.md — Shared Facts, Cross-Reference Map, TODO Registry, Section Numbering

## Step 2: Your Assigned Documents
[LIST — e.g.:]
- 01_ENVIRONMENT.md (Task ID: 1)
- 02_DEPENDENCIES.md (Task ID: 2)
- 03_ARCHITECTURE.md (Task ID: 3)
- 04_STRUCTURE.md (Task ID: 4)

## Step 3: For Each Document
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read relevant source files (identify from skeleton + your own exploration)
3. Write ai-docs/XX_FILE.md following:
   - SPEC rules (evidence-based, tables over prose)
   - Skeleton's Shared Facts — BOTH Value Facts AND Classification Map (do NOT re-derive independently)
   - Skeleton's Section Numbering (§ numbers exactly as assigned)
   - Skeleton's Cross-Reference Map (canonical = full detail, mention = one-line + → ref)
   - ## Evidence at end
   - Metadata: > Generated: YYYY-MM-DD | Based on commit: <hash from skeleton>
4. TaskUpdate({ "taskId": "<id>", "status": "completed" })

## Conflict Escalation Rule
If you encounter a cross-cutting fact (component category, rendering mode, config flag, etc.)
that is NOT in the skeleton's Shared Facts or Classification Map, do NOT guess.
Flag it in your completion message so the orchestrator or reviewer can resolve it.

## Step 4: When All Done
SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Completed: [files]. Notes: [observations]. Unresolved classifications: [list or 'none']",
  "summary": "Docs XX-YY complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

### 2-4. Monitor & Shutdown Write Team

```json
TaskList()  // check progress periodically
```

**Timeout & retry policy:**
- Check progress every 2-3 minutes via TaskList()
- If an agent's task stays `in_progress` for >10 minutes with no file output, send a status check message
- If still stuck after another 5 minutes, note the failure and proceed with remaining tasks
- After all other tasks complete, attempt the failed task by spawning a replacement agent
- If replacement also fails, the orchestrator writes the document directly as fallback

When all 11 tasks completed:

```json
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-3", "content": "All tasks complete" })

// After all confirm shutdown
// Uses current team context — no need to specify name
TeamDelete()
```

---

## Step 3: Review Team (ai-docs-review)

### 3-1. Create Team & Tasks

```json
// Generate unique team name to prevent collision with stale teams from crashed runs
// Format: ai-docs-review-{YYYYMMDD-HHmmss} (e.g., ai-docs-review-20260220-144500)
TeamCreate({ "team_name": "ai-docs-review-{timestamp}", "description": "AI docs review and verification" })

// Reviewer tasks (parallel)
TaskCreate({ "subject": "Review docs 01-04", "description": "Deep review + fix 01_ENVIRONMENT, 02_DEPENDENCIES, 03_ARCHITECTURE, 04_STRUCTURE", "activeForm": "Reviewing docs 01-04" })
TaskCreate({ "subject": "Review docs 05-07", "description": "Deep review + fix 05_DATA_MODELS, 06_API, 07_BUSINESS_LOGIC", "activeForm": "Reviewing docs 05-07" })
TaskCreate({ "subject": "Review docs 08-11", "description": "Deep review + fix 08_DEBUG, 09_STANDARDS, 10_WARNINGS, 11_TODO", "activeForm": "Reviewing docs 08-11" })

// Cross-checker task (blocked by reviewers)
TaskCreate({ "subject": "Verify consistency + write INDEX", "description": "Check §/cross-ref/shared-facts across all docs, write 00_INDEX.md", "activeForm": "Verifying consistency" })

// Set dependency: cross-checker waits for all reviewers
TaskUpdate({ "taskId": "4", "addBlockedBy": ["1", "2", "3"] })
```

### 3-2. Assign Tasks

> **Note**: Task IDs shown below (1-4) assume sequential creation within a fresh team.
> If the Team API assigns different IDs, use the actual IDs returned by TaskCreate.
> The orchestrator should track the mapping: document name → taskId.

```json
TaskUpdate({ "taskId": "1", "owner": "reviewer-1" })
TaskUpdate({ "taskId": "2", "owner": "reviewer-2" })
TaskUpdate({ "taskId": "3", "owner": "reviewer-3" })
TaskUpdate({ "taskId": "4", "owner": "cross-checker" })
```

### 3-3. Spawn Reviewers (parallel, single message)

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-1",
  "description": "Review and fix docs 01-04",
  "prompt": "<Reviewer Prompt — assigned: 01, 02, 03, 04>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-2",
  "description": "Review and fix docs 05-07",
  "prompt": "<Reviewer Prompt — assigned: 05, 06, 07>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-3",
  "description": "Review and fix docs 08-11",
  "prompt": "<Reviewer Prompt — assigned: 08, 09, 10, 11>"
})
```

### Reviewer Prompt Template

```
You are a documentation reviewer agent in the "ai-docs-review" team.

Your job is to deeply review and improve existing documentation files.
The initial drafts were written by another team — expect ~65-80% completeness.
Your goal is to bring them to ~95%+ by finding and fixing gaps.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (your assigned docs)
2. ai-docs/.skeleton.md — Shared Facts, Cross-Reference Map, TODO Registry, Section Numbering

## Step 2: Your Assigned Documents
[LIST — e.g.:]
- ai-docs/01_ENVIRONMENT.md (Task ID: 1)
- ai-docs/02_DEPENDENCIES.md (Task ID: 2)
- ai-docs/03_ARCHITECTURE.md (Task ID: 3)
- ai-docs/04_STRUCTURE.md (Task ID: 4)

## Step 3: For Each Document
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read the existing document thoroughly
3. Read the ACTUAL SOURCE FILES referenced in the document's ## Evidence section
   — AND search for additional source files the writer may have missed
4. Check against SPEC §5 requirements for this document type:
   - Are all mandatory sections present?
   - Are all items covered? (e.g., every endpoint in 06, every table in 05, every env var in 01)
   - Are descriptions accurate against the actual source code?
   - Are cross-references valid and following the skeleton's map?
   - Are § numbers matching the skeleton?
   - Is the ## Evidence section complete?
   - **Classification check**: Does the document classify any file/component/module?
     If yes, verify against the skeleton's Classification Map AND the actual source file.
     If the document contradicts the skeleton, fix the document.
5. Fix all issues using the Edit tool:
   - Add missing content (endpoints, fields, env vars, commands, etc.)
   - Correct inaccuracies (especially classification mismatches — server/client, static/dynamic, etc.)
   - Add missing cross-references
   - Expand thin sections that lack detail
   - Ensure evidence section lists ALL files you referenced
6. TaskUpdate({ "taskId": "<id>", "status": "completed" })

## Review Checklist (per document)
- [ ] All SPEC §5 mandatory sections present
- [ ] Content matches actual source code (no stale/wrong info)
- [ ] No content gaps (missing endpoints, fields, env vars, etc.)
- [ ] Cross-references valid (canonical/mention per skeleton)
- [ ] § numbers match skeleton
- [ ] ## Evidence is complete and accurate
- [ ] Metadata line present (Generated date + commit hash)
- [ ] No speculation — only observable facts

## Step 4: When All Done
SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Review complete for [files].\n\nPer-doc summary:\n- 01: [changes made]\n- 02: [changes made]\n...",
  "summary": "Review 01-04 complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

### 3-4. After Reviewers Complete → Spawn Cross-Checker

When all 3 reviewer tasks are `completed`:

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "cross-checker",
  "description": "Verify consistency and write INDEX",
  "prompt": "<Cross-Checker Prompt>"
})
```

### Cross-Checker Prompt

```
You are the cross-checker agent in the "ai-docs-review" team.

Your job is structural/metadata verification across all documents + writing 00_INDEX.md.

For structural checks (§ numbers, cross-refs, evidence, metadata): you do NOT need to deeply read body content.
However, writing 00_INDEX.md REQUIRES reading specific documents for content:
- 01_ENVIRONMENT.md — needed for Quick Start (§0-2)
- 02_DEPENDENCIES.md — needed for Quick Start prerequisites (§0-2)
- 10_WARNINGS.md — needed for Critical Warnings Summary (§0-4)
Read these three documents thoroughly before writing the INDEX.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0-6 (Verification checklist) and §3 (INDEX rules)
2. ai-docs/.skeleton.md — for cross-checking

## Step 2: Verify Each Document (01 through 11)
Apply the SPEC §0-6 verification checklist to each document. The 8 items are:
1. File exists with correct filename
2. All §N section headers match skeleton's Section Numbering Scheme
3. `## Evidence` section exists listing source files
4. Cross-references (`→ Detail:`) resolve to existing sections
5. Shared Facts (Value Facts + Classification Map) match skeleton
6. Metadata line present (`> Generated: ...`)
7. Non-ASCII encoding integrity (CJK, box-drawing chars intact)
8. 09_STANDARDS anti-patterns reference 11_TODO IDs where applicable

Additionally verify:
9. 11_TODO includes all items from skeleton's TODO Registry
10. Quick Start in 00_INDEX (once written) is logically executable

**Classification consistency check**: For every file listed in the skeleton's Classification Map,
verify that ALL documents mentioning that file use the SAME classification.
If two documents disagree (e.g., one calls a component "server", another calls it "client"),
check the actual source file and fix ALL incorrect documents.

**Cross-partition consistency check**: Writers work in partitions (01-04, 05-07, 08-11).
Cross-partition contradictions (e.g., 03_ARCHITECTURE vs 05_DATA_MODELS describing the same flow differently)
are NOT caught by reviewers since they review the same partition. You MUST:
- Scan for overlapping topics across partition boundaries
- Verify that cross-references between partitions resolve correctly
- Fix any factual contradictions between documents from different partitions

Fix any structural issues with Edit tool.

## Step 3: Write 00_INDEX.md
Write ai-docs/00_INDEX.md following SPEC §3:
- §0-1 Project Identity table
- §0-2 Quick Start (minimum command sequence, all steps explicit, no secrets)
- §0-3 Document Map with Common Task Routing table
- §0-4 Critical Warnings Summary (3-5 lines, → 10_WARNINGS.md §N)

NOTE: INDEX uses the §0-N namespace to avoid collision with 03_ARCHITECTURE's §3-N.

## Step 4: Self-Verify 00_INDEX.md
Apply the same 8-item checklist from Step 2 to your own 00_INDEX.md:
- [ ] All §0-N section headers present
- [ ] ## Evidence section exists (list ALL 11 docs + skeleton + any source files you read)
- [ ] Metadata line present
- [ ] No speculation — only observable facts
The cross-checker has no external reviewer — you must verify your own output.

## Step 5: Final Report
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Cross-check complete.\n\n## 8-Item Checklist (per doc)\n[results table]\n\n## Classification Consistency\n[conflicts found and resolved, or 'none']\n\nFixes applied: [list or 'none']\n00_INDEX.md written and self-verified.",
  "summary": "Cross-check and INDEX complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

**Timeout & retry policy (review team):**
- Check progress every 2-3 minutes via TaskList()
- If a reviewer's task stays `in_progress` for >10 minutes with no file changes, send a status check message
- If still stuck after another 5 minutes, note the failure and proceed with remaining tasks
- After all other reviewer tasks complete, attempt the failed task by spawning a replacement reviewer
- If replacement also fails, the orchestrator performs the review directly as fallback
- The cross-checker only spawns after all 3 reviewer tasks are `completed`

### 3-5. Shutdown Review Team

After cross-checker reports:

```json
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-3", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "cross-checker", "content": "All tasks complete" })

// Uses current team context — no need to specify name
TeamDelete()
```

---

## Step 4: Report to User

Summarize:
- 12 files generated in `ai-docs/`
- Cross-checker verification results
- Any notable issues or observations from reviewers

---

## Update Mode

When user selects "Update" in Step 0, the following pipeline runs.

**Key differences from Full Generation:**
- No from-scratch writing — all changes are Edit-based on existing documents
- Audit Team replaces Scout — audits existing docs vs actual code, produces refactoring list
- Information flows through `.update-manifest.md` (temporary, deleted after completion)
- Writer/Reviewer count scales dynamically based on affected document count

---

### Step U-1: Audit Team (ai-docs-audit)

The Audit Team replaces the single Scout agent. Skeleton quality determines overall documentation quality — a team of parallel auditors ensures thorough coverage even for large codebases.

#### U-1.1 Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-audit-{timestamp}", "description": "Audit existing docs vs code changes" })

// Phase A: Parallel auditors
TaskCreate({ "subject": "Audit docs 01-04", "description": "Audit 01_ENVIRONMENT, 02_DEPENDENCIES, 03_ARCHITECTURE, 04_STRUCTURE against actual code + git diff", "activeForm": "Auditing docs 01-04" })
TaskCreate({ "subject": "Audit docs 05-07", "description": "Audit 05_DATA_MODELS, 06_API, 07_BUSINESS_LOGIC against actual code + git diff", "activeForm": "Auditing docs 05-07" })
TaskCreate({ "subject": "Audit docs 08-11", "description": "Audit 08_DEBUG, 09_STANDARDS, 10_WARNINGS, 11_TODO against actual code + git diff", "activeForm": "Auditing docs 08-11" })

// Phase B: Synthesizer (blocked by all auditors)
TaskCreate({ "subject": "Synthesize audit reports", "description": "Merge 3 audit reports → update skeleton + generate manifest + refactoring list", "activeForm": "Synthesizing audit results" })

TaskUpdate({ "taskId": "4", "addBlockedBy": ["1", "2", "3"] })
```

#### U-1.2 Assign & Spawn Auditors (parallel)

```json
TaskUpdate({ "taskId": "1", "owner": "auditor-1" })
TaskUpdate({ "taskId": "2", "owner": "auditor-2" })
TaskUpdate({ "taskId": "3", "owner": "auditor-3" })
TaskUpdate({ "taskId": "4", "owner": "synthesizer" })
```

Spawn all 3 auditors in a single message (parallel execution):

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-1",
  "description": "Audit docs 01-04 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 01, 02, 03, 04>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-2",
  "description": "Audit docs 05-07 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 05, 06, 07>"
})
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-3",
  "description": "Audit docs 08-11 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 08, 09, 10, 11>"
})
```

#### Auditor Prompt Template

```
You are an auditor agent in the "ai-docs-audit" team.

Your job is to compare existing AI documentation against the ACTUAL source code
and identify everything that needs updating — both from code changes AND doc-internal issues.

## Context
- Previous commit: [PREV_COMMIT]
- Current commit: [CURRENT_COMMIT]
- Changed files (git diff): [CHANGED_FILES_LIST]

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (your assigned docs)
2. ai-docs/.skeleton.md — Shared Facts, Cross-Reference Map, TODO Registry
3. Your assigned documents (existing ai-docs)

## Step 2: Your Assigned Documents
[LIST — e.g.:]
- ai-docs/01_ENVIRONMENT.md (Task ID: 1)
- ai-docs/02_DEPENDENCIES.md (Task ID: 1)
- ai-docs/03_ARCHITECTURE.md (Task ID: 1)
- ai-docs/04_STRUCTURE.md (Task ID: 1)

## Step 3: For Each Document
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read the existing document thoroughly
3. Read ALL source files listed in the document's ## Evidence section
4. Additionally, explore source files related to git diff changes in your document's domain
5. Identify discrepancies in two categories:

### A. Code-Driven Updates (from git diff)
Changes in source code that the document should reflect but currently doesn't:
- New files/modules/endpoints not documented
- Modified behavior not reflected in docs
- Renamed/moved/deleted items still referenced
- Changed configuration, env vars, dependencies

### B. Document-Internal Issues (independent of git diff)
Problems in the document itself, even if the code hasn't changed:
- Stale information (references to files/functions that no longer exist)
- Broken cross-references (→ Detail: pointing to non-existent sections)
- SPEC §5 violations (missing mandatory sections, wrong format)
- Skeleton inconsistencies (Shared Facts, Classification Map mismatches)
- Incomplete Evidence section (missing source files that were clearly read)
- Thin sections that lack adequate detail
- Inaccurate descriptions (code does X but doc says Y)

## Step 4: Write Audit Report
Write your findings to: ai-docs/.audit-report-N.md (N = your auditor number: 1, 2, or 3)

Report format:
```markdown
# Audit Report N — Docs [range]
> Auditor: auditor-N | Commit range: [PREV]..[CURRENT]

## Per-Document Findings

### [XX_FILENAME.md]
#### Code-Driven Updates
| Section | Source File | Issue | Priority |
|---------|-----------|-------|----------|
(P0=critical, P1=important, P2=moderate, P3=minor)

#### Document-Internal Issues
| Section | Issue | Priority |
|---------|-------|----------|

#### Skeleton Corrections
- Shared Facts: [corrections needed, or "none"]
- Cross-Ref Map: [corrections needed, or "none"]
- TODO Registry: [new items to add, or "none"]

### [next document...]
(repeat for each assigned document)
```

## Step 5: Report Completion
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Audit complete for docs [range].\n\nSummary: [N] code-driven updates, [M] doc-internal issues found.\nCritical items: [list or 'none']",
  "summary": "Audit [range] complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### U-1.3 After Auditors Complete → Spawn Synthesizer

When all 3 auditor tasks are `completed`:

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "synthesizer",
  "description": "Merge audit reports and generate manifest",
  "prompt": "<Synthesizer Prompt>"
})
```

#### Synthesizer Prompt

```
You are the synthesizer agent in the "ai-docs-audit" team.

Your job is to merge the 3 auditor reports into a coherent update plan:
update the skeleton, and generate the update manifest with a refactoring list.

## Step 1: Read These Files
1. ai-docs/.audit-report-1.md
2. ai-docs/.audit-report-2.md
3. ai-docs/.audit-report-3.md
4. ai-docs/.skeleton.md (current)
5. ai-docs/SPEC.md — §0-2 (for skeleton structure rules)

## Step 2: Update Skeleton (in-place Edit)
Based on the 3 audit reports, update ai-docs/.skeleton.md:
- **Shared Facts**: Update Value Facts and Classification Map with corrections from auditors
- **Cross-Reference Map**: Fix broken references, add new ones
- **TODO Registry**: Add new items discovered by auditors (use EXTRA-N IDs)
- **Section Numbering**: Adjust if auditors identified structural changes needed
- **File-to-Document Map**: Update with new/renamed/deleted files

Use the Edit tool — do NOT rewrite the entire skeleton. Preserve unchanged sections.

## Step 3: Generate Update Manifest
Write to: ai-docs/.update-manifest.md

```markdown
# Update Manifest
> Previous: [PREV_COMMIT] | Current: [CURRENT_COMMIT] | Generated: YYYY-MM-DD

## Refactoring List

### Code-Driven Updates
| Affected Doc | Section | Source File | Issue | Priority |
|-------------|---------|------------|-------|----------|
(Merged and deduplicated from all 3 audit reports)

### Document-Internal Issues
| Affected Doc | Section | Issue | Priority |
|-------------|---------|-------|----------|
(Merged and deduplicated from all 3 audit reports)

### Summary
- Total affected documents: N / 11
- Code-driven updates: N items
- Document-internal issues: N items
- Critical (P0): N items
- Documents requiring NO changes: [list]

## Update Instructions (per affected document)

### [XX_FILENAME.md]
Specific edit instructions for writers:
- §N-M: [what to change and why]
- §N-M: [what to change and why]
- Evidence: [files to add/remove]
- Metadata: update commit hash to [CURRENT_COMMIT]

### [next document...]
(repeat for each affected document, ordered by document number)

## Skeleton Changes Applied
- Shared Facts: [summary of changes]
- Cross-Ref Map: [summary of changes]
- TODO Registry: [new items added]
```

## Step 4: Resolve Conflicts
If auditor reports contradict each other (e.g., different classification for the same file):
- Check the actual source file to determine the correct answer
- Apply the correction to the skeleton
- Note the resolution in the manifest

## Step 5: Report Completion
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Synthesis complete.\n\nAffected docs: [list]\nRefactoring items: [N] code-driven + [M] doc-internal\nSkeleton updated. Manifest written to .update-manifest.md",
  "summary": "Synthesis complete — N docs affected"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### U-1.4 Shutdown Audit Team + User Confirmation

After synthesizer reports:

```json
SendMessage({ "type": "shutdown_request", "recipient": "auditor-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "auditor-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "auditor-3", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "synthesizer", "content": "All tasks complete" })
TeamDelete()
```

**Orchestrator action after Audit Team shutdown:**

1. Read `ai-docs/.update-manifest.md` — specifically the `## Refactoring List` and `## Summary` sections
2. Present the refactoring list to the user:
   - Number of affected documents
   - Code-driven updates (with priorities)
   - Document-internal issues (with priorities)
   - Documents requiring no changes
3. Ask user: **"Proceed with these updates?"**
   - User may remove items or adjust scope
   - If user approves: proceed to Step U-2
   - If user cancels: clean up temp files and stop

---

### Step U-2: Update Write Team (ai-docs-update)

#### Writer Scaling

| Affected Docs | Writers | Assignment |
|--------------|---------|------------|
| 1–2 | 1 writer | All docs to writer-1 |
| 3–5 | 2 writers | Split evenly |
| 6+ | 3 writers | Split by partition (01-04, 05-07, 08-11) |

#### U-2.1 Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-update-{timestamp}", "description": "Update affected documents" })

// Create one task per affected document
TaskCreate({ "subject": "Update XX_FILENAME.md", "description": "Edit per manifest instructions", "activeForm": "Updating XX_FILENAME.md" })
// ... repeat for each affected document
```

#### U-2.2 Assign & Spawn Writers

Assign tasks to writers based on scaling rules above. Spawn all writers in parallel.

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-update-{timestamp}",
  "name": "update-writer-1",
  "description": "Update docs [list]",
  "prompt": "<Update Writer Prompt — assigned: [doc list]>"
})
// ... additional writers if needed
```

#### Update Writer Prompt Template

```
You are an update writer agent in the "ai-docs-update" team.

Your job is to EDIT existing documentation files based on the update manifest.
You are NOT writing from scratch — you are surgically modifying existing documents.

## Critical Rules
1. Use the Edit tool for all changes — NEVER use Write to overwrite entire documents
2. Preserve all unchanged sections exactly as they are
3. Follow the manifest's per-document instructions precisely
4. Update the metadata line (date + commit hash) for every document you modify
5. Update the ## Evidence section if you reference new source files

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (your assigned docs)
2. ai-docs/.skeleton.md (already updated by synthesizer)
3. ai-docs/.update-manifest.md — specifically your assigned documents' instructions
4. Your assigned existing documents

## Step 2: Your Assigned Documents
[LIST — e.g.:]
- ai-docs/03_ARCHITECTURE.md (Task ID: 1)
- ai-docs/07_BUSINESS_LOGIC.md (Task ID: 2)

## Step 3: For Each Document
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read the manifest's "Update Instructions" for this document
3. Read the existing document
4. Read the relevant source files to verify the changes you need to make
5. Apply each edit instruction:
   - For content updates: Edit the specific section with corrected information
   - For new sections: Insert at the correct position per skeleton's Section Numbering
   - For deleted content: Remove cleanly, no placeholder comments
   - For cross-ref fixes: Update the reference target
6. Update metadata line: `> Generated: YYYY-MM-DD | Based on commit: [CURRENT_COMMIT]`
7. Update ## Evidence section: add any new source files you referenced, remove deleted ones
8. TaskUpdate({ "taskId": "<id>", "status": "completed" })

## Conflict Escalation
If a manifest instruction seems wrong after reading the actual source code:
- Do NOT blindly follow the instruction
- Check the source code yourself
- Apply the correct change based on evidence
- Flag the discrepancy in your completion message

## Step 4: When All Done
SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Updated: [files].\n\nPer-doc summary:\n- XX: [sections modified]\n- YY: [sections modified]\nEscalations: [list or 'none']",
  "summary": "Update [doc list] complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### U-2.3 Monitor & Shutdown Write Team

**Timeout & retry policy:**
- Check progress every 2-3 minutes via TaskList()
- If an agent's task stays `in_progress` for >10 minutes with no file changes, send a status check message
- If still stuck after another 5 minutes, note the failure and proceed with remaining tasks
- After all other tasks complete, attempt the failed task by spawning a replacement agent
- If replacement also fails, the orchestrator writes the document directly as fallback

When all tasks completed:

```json
SendMessage({ "type": "shutdown_request", "recipient": "update-writer-1", "content": "All tasks complete" })
// ... for each writer
TeamDelete()
```

---

### Step U-3: Update Review Team (ai-docs-update-review)

#### U-3.1 Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-update-review-{timestamp}", "description": "Review updated documents" })

TaskCreate({ "subject": "Review updated documents", "description": "Deep review all modified docs against source code + manifest", "activeForm": "Reviewing updated docs" })
TaskCreate({ "subject": "Cross-check consistency", "description": "Verify cross-refs between updated and unchanged docs + update 00_INDEX.md if needed", "activeForm": "Cross-checking consistency" })

// Cross-checker waits for reviewer
TaskUpdate({ "taskId": "2", "addBlockedBy": ["1"] })
```

#### U-3.2 Spawn Reviewer

```json
TaskUpdate({ "taskId": "1", "owner": "update-reviewer" })
TaskUpdate({ "taskId": "2", "owner": "cross-checker" })

Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-update-review-{timestamp}",
  "name": "update-reviewer",
  "description": "Review all updated documents",
  "prompt": "<Update Reviewer Prompt>"
})
```

#### Update Reviewer Prompt Template

```
You are the update reviewer agent in the "ai-docs-update-review" team.

Your job is to deeply review the documents that were just modified by the update writers.
Verify accuracy against actual source code and ensure manifest instructions were followed correctly.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (modified docs)
2. ai-docs/.skeleton.md (updated)
3. ai-docs/.update-manifest.md — the instructions that writers followed

## Step 2: Modified Documents
[LIST of all documents modified in Step U-2]

## Step 3: For Each Modified Document
1. TaskUpdate({ "taskId": "1", "status": "in_progress" })
2. Read the updated document
3. Read the manifest instructions for this document
4. Verify EACH manifest instruction was applied correctly:
   - Was the change made accurately?
   - Does it match the actual source code? (read the relevant source files)
   - Were any instructions missed?
5. Apply the full review checklist (same as Full Generation reviewers):
   - [ ] All SPEC §5 mandatory sections present
   - [ ] Content matches actual source code
   - [ ] No content gaps
   - [ ] Cross-references valid
   - [ ] § numbers match skeleton
   - [ ] ## Evidence is complete and accurate
   - [ ] Metadata line updated (new date + commit hash)
   - [ ] No speculation — only observable facts
6. Fix any issues using Edit tool

## Step 4: When All Done
TaskUpdate({ "taskId": "1", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Review complete.\n\nPer-doc summary:\n- XX: [issues found and fixed]\n- YY: [issues found and fixed]\nOverall quality: [assessment]",
  "summary": "Update review complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### U-3.3 After Reviewer → Spawn Cross-Checker

```json
Task({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-update-review-{timestamp}",
  "name": "cross-checker",
  "description": "Cross-check consistency and update INDEX",
  "prompt": "<Update Cross-Checker Prompt>"
})
```

#### Update Cross-Checker Prompt

```
You are the cross-checker agent in the "ai-docs-update-review" team.

Your job is to verify consistency between UPDATED and UNCHANGED documents,
then update 00_INDEX.md if needed.

This is critical: update writers only touched some documents. The unchanged documents
may now have stale cross-references pointing to updated content, or updated documents
may reference unchanged sections that no longer align.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0-6 (verification checklist) and §3 (INDEX rules)
2. ai-docs/.skeleton.md (updated)
3. ai-docs/.update-manifest.md — to know which docs were modified

## Step 2: Cross-Partition Consistency Check

For each UPDATED document:
1. Find all cross-references TO other documents (→ Detail: XX_FILE.md §N)
2. Verify the target section exists and the reference is still accurate
3. Find all UNCHANGED documents that reference the updated document
4. Verify those references still point to valid, accurate content

For Classification consistency:
- Check skeleton's Classification Map
- Verify ALL documents (updated and unchanged) use the same classifications

Fix any issues with Edit tool.

## Step 3: Update 00_INDEX.md (if needed)

Read 00_INDEX.md. Determine if updates are needed:
- §0-1 Project Identity: update if project metadata changed
- §0-2 Quick Start: update if environment/setup changed
- §0-3 Document Map: update if document scope changed
- §0-4 Critical Warnings Summary: update if warnings changed

If 00_INDEX.md needs changes:
- Edit specific sections (do NOT rewrite from scratch)
- Update metadata line (date + commit hash)

If 00_INDEX.md needs NO changes:
- Still update the metadata commit hash (the docs are now based on a newer commit)

## Step 4: Self-Verify
Apply the 8-item SPEC §0-6 verification checklist to 00_INDEX.md.

## Step 5: Final Report
TaskUpdate({ "taskId": "2", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Cross-check complete.\n\nCross-ref issues found: [N]\nClassification conflicts: [N]\nFixes applied: [list or 'none']\n00_INDEX.md: [updated/no changes needed]",
  "summary": "Cross-check complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### U-3.4 Shutdown Review Team

```json
SendMessage({ "type": "shutdown_request", "recipient": "update-reviewer", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "cross-checker", "content": "All tasks complete" })
TeamDelete()
```

---

### Step U-4: Cleanup + Report

1. **Delete temporary files**:
   - `ai-docs/.audit-report-1.md`
   - `ai-docs/.audit-report-2.md`
   - `ai-docs/.audit-report-3.md`
   - `ai-docs/.update-manifest.md`
2. **Report to user**:
   - Documents updated: [list]
   - Documents unchanged: [list]
   - Cross-checker results
   - Any unresolved issues or escalations

---

## §DEFAULT-SPEC

If `ai-docs/SPEC.md` does not exist, create it before proceeding.

**Recommended approach**: If this repository (ai-docs) is available, copy the canonical `SPEC.md` from it. This ensures consistency and completeness.

**Fallback** (if canonical SPEC is not available): Generate a SPEC covering these sections:

1. **§0 Hard Rules**: Evidence-based writing, Evidence sections, output format (UTF-8, Mermaid fences, code blocks), metadata (date + commit hash), cross-reference principle (canonical source priority table), verification checklist (the 8 items below), INDEX last, security (no secrets), section numbering (§N format). The 8-item verification checklist that reviewers and cross-checker must apply to every document:
   1. File exists with correct filename
   2. All §N section headers match skeleton's numbering scheme
   3. `## Evidence` section exists listing source files
   4. Cross-references (`→ Detail:`) resolve to existing sections
   5. Shared Facts (Value Facts + Classification Map) match skeleton
   6. Metadata line present (`> Generated: ...`)
   7. Anti-patterns in 09 reference TODO IDs from 11
   8. TODO registry in 11 includes all skeleton items
2. **§1 Execution Model**: 3-Phase (scout → write team → review team), Phase 0 skeleton structure (Value Facts, Classification Map, Cross-Reference Map, TODO Registry with ID rules, Section Numbering), Phase 1 parallel writing rules (writers must use skeleton classifications verbatim — never re-derive), Phase 2 parallel review + cross-check (cross-checker must self-verify their own 00_INDEX output)
3. **§2 File List**: 12 mandatory files (00_INDEX through 11_TODO) with phase assignment
4. **§3 INDEX Rules**: Project Identity, Quick Start (explicit all steps), Document Map with Common Task Routing (8 default rows), Critical Warnings Summary (3-5 lines with cross-refs). INDEX uses §0-N namespace (§0-1 through §0-4) to avoid collision with 03_ARCHITECTURE's §3-N
5. **§4 Common Principles**: AI audience, format priority (table > bullet > code > prose), N/A handling, Evidence, empty item consolidation, directory tree exclusions, section numbering compliance, ownership principle
6. **§5 Per-Document Content**: Detailed requirements for all 11 content documents including ownership restrictions (04, 06), boundary rules (06↔07), TODO Registry compliance (09, 11), priority system (P0-P3 with tiebreaker)
7. **§6 Update Policy**: Regeneration triggers, method selection (full vs update mode), incremental update constraints, temporary file cleanup

Generate the SPEC as a self-contained, project-agnostic document. Write it to `ai-docs/SPEC.md`.
