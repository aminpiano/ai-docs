---
name: ai-project-docs
description: |
  Generate AI-optimized project documentation using agent teams for parallel generation.
  Creates structured, token-efficient docs that enable ANY AI to immediately work on the project.

  Uses Claude Code's agent team feature: scout → write team → review team.
  Main session acts as lightweight orchestrator — all heavy work delegated to agents via files.

  Use when: (1) Setting up a new project for AI collaboration, (2) User asks to "document this project for AI", (3) Creating architecture docs, (4) Making project AI-readable, (5) "Create AI docs", "generate project map", "make this AI-friendly"
---

# AI Project Documentation Generator (Team Edition)

Generate 12 AI-optimized documentation files in `ai-docs/` using agent teams.

**Architecture**: Main session is a lightweight orchestrator. All heavy lifting happens in spawned agents. Information flows through files, not the main session's context.

```
Main Session (Orchestrator)
  │
  ├─ Step 0: Ensure ai-docs/SPEC.md exists
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

## Step 0: Ensure SPEC Exists

Check if `ai-docs/SPEC.md` exists.
- **If exists**: proceed.
- **If not exists**: generate it per §DEFAULT-SPEC (end of this document), then proceed.

This is the only file the main session reads directly.

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
   - Explicit TODOs/FIXMEs (Grep sweep for TODO, FIXME, HACK, BUG, WORKAROUND)
   - Implicit issues discovered during analysis
3. Generate the skeleton per SPEC §1-2:
   A. Shared Facts
   B. Cross-Reference Map
   C. TODO Registry (pre-assigned IDs)
   D. Section Numbering Scheme (all 11 documents)
4. Write to: ai-docs/.skeleton.md
5. Constraint: ~200 lines max. Metadata only.

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]

## When Done
Report completion with one-line summary (tech stack, file count, etc.).
```

**After scout completes**: confirm `ai-docs/.skeleton.md` exists. Do NOT read it.

---

## Step 2: Write Team (ai-docs-gen)

### 2-1. Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-gen", "description": "AI docs parallel writing" })

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
   - Skeleton's Shared Facts (do NOT re-derive independently)
   - Skeleton's Section Numbering (§ numbers exactly as assigned)
   - Skeleton's Cross-Reference Map (canonical = full detail, mention = one-line + → ref)
   - ## Evidence at end
   - Metadata: > Generated: YYYY-MM-DD | Based on commit: <hash from skeleton>
4. TaskUpdate({ "taskId": "<id>", "status": "completed" })

## Step 4: When All Done
SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Completed: [files]. Notes: [observations]",
  "summary": "Docs XX-YY complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

### 2-4. Monitor & Shutdown Write Team

```json
TaskList()  // check progress periodically
```

When all 11 tasks completed:

```json
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "doc-writer-3", "content": "All tasks complete" })

// After all confirm shutdown
TeamDelete()
```

---

## Step 3: Review Team (ai-docs-review)

### 3-1. Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-review", "description": "AI docs review and verification" })

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
5. Fix all issues using the Edit tool:
   - Add missing content (endpoints, fields, env vars, commands, etc.)
   - Correct inaccuracies
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
You do NOT need to deeply read document body content — focus on structure and metadata.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0-6 (Verification checklist) and §3 (INDEX rules)
2. ai-docs/.skeleton.md — for cross-checking

## Step 2: For Each Document (01 through 11)
Read only enough to verify:
- [ ] File exists
- [ ] All §N section headers present (match skeleton's Section Numbering Scheme)
- [ ] ## Evidence section exists
- [ ] Cross-references (`→ Detail: XX_FILE.md §N`) point to sections that actually exist
- [ ] Shared Facts (versions, paths, commit hash) match skeleton across all documents
- [ ] Metadata line present: `> Generated: YYYY-MM-DD | Based on commit: <hash>`
- [ ] 09_STANDARDS anti-patterns reference 11_TODO IDs where applicable
- [ ] 11_TODO includes all items from skeleton's TODO Registry

Fix any structural issues with Edit tool.

## Step 3: Write 00_INDEX.md
Write ai-docs/00_INDEX.md following SPEC §3:
- §3-1 Project Identity table
- §3-2 Quick Start (minimum command sequence, all steps explicit, no secrets)
- §3-3 Document Map with Common Task Routing table
- §3-4 Critical Warnings Summary (3-5 lines, → 10_WARNINGS.md §N)

## Step 4: Final Verification Output
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Cross-check complete.\n\n## Verification Checklist\n- [x/] item...\n\nFixes applied: [list or 'none']\n00_INDEX.md written.",
  "summary": "Cross-check and INDEX complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

### 3-5. Shutdown Review Team

After cross-checker reports:

```json
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "reviewer-3", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "cross-checker", "content": "All tasks complete" })

TeamDelete()
```

---

## Step 4: Report to User

Summarize:
- 12 files generated in `ai-docs/`
- Cross-checker verification results
- Any notable issues or observations from reviewers

---

## Updating Existing Docs

- 1–2 docs affected: update directly, no team needed
- 3+ docs: run full process (SPEC §6) — write team + review team
- Always regenerate 00_INDEX.md last

---

## §DEFAULT-SPEC

If `ai-docs/SPEC.md` does not exist, create it before proceeding. The default SPEC must cover:

1. **§0 Hard Rules**: Evidence-based writing, Evidence sections, output format (UTF-8, Mermaid fences, code blocks), metadata (date + commit hash), cross-reference principle (canonical source priority table), verification checklist (8 items), INDEX last, security (no secrets), section numbering (§N format)
2. **§1 Execution Model**: 3-Phase (scout → write team → review team), Phase 0 skeleton structure (Shared Facts, Cross-Reference Map, TODO Registry with ID rules, Section Numbering), Phase 1 parallel writing rules, Phase 2 parallel review + cross-check
3. **§2 File List**: 12 mandatory files (00_INDEX through 11_TODO) with phase assignment
4. **§3 INDEX Rules**: Project Identity, Quick Start (explicit all steps), Document Map with Common Task Routing (8 default rows), Critical Warnings Summary (3-5 lines with cross-refs)
5. **§4 Common Principles**: AI audience, format priority (table > bullet > code > prose), N/A handling, Evidence, empty item consolidation, directory tree exclusions, section numbering compliance, ownership principle
6. **§5 Per-Document Content**: Detailed requirements for all 11 content documents including ownership restrictions (04, 06), boundary rules (06↔07), TODO Registry compliance (09, 11), priority system (P0-P3 with tiebreaker)
7. **§6 Update Policy**: Regeneration triggers, full-regeneration default, diff verification

Generate the SPEC as a self-contained, project-agnostic document. Write it to `ai-docs/SPEC.md`.
