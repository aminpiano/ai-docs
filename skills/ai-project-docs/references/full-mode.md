## Step 1: Scout Team → Skeleton File (ai-docs-scout)

Parallel scout team explores the codebase by domain partition, then a synthesizer merges findings into the skeleton. Main session does NOT read source code.

**Partitioning is load-based, NOT sequential.** Documents vary wildly in source code volume. The default partition below is only a bootstrap partition for scouting. The synthesizer must use scout workload signals to size the later writer and reviewer teams. Do not carry the 4-scout partition forward as the writer/reviewer partition.

The scout phase must answer two questions:
1. What shared facts should all documents use?
2. How much writer/reviewer capacity does each final document need?

#### Default Scout Partition (4 scouts)

| Scout | Domain | Target Documents | Rationale |
|-------|--------|-----------------|-----------|
| scout-1 | **Business logic** (services, core modules, workflows) | 07_BUSINESS_LOGIC | Heaviest domain (~14k lines). Must be solo. |
| scout-2 | **Data + API** (models, schemas, routes, endpoints) | 05_DATA_MODELS, 06_API | Data + API layer share schemas. ~11k lines combined. |
| scout-3 | **Infrastructure** (env, deps, architecture, structure, config) | 01_ENVIRONMENT, 02_DEPENDENCIES, 03_ARCHITECTURE, 04_STRUCTURE, 11_TODO | Infrastructure + structure + todos. ~10k lines combined. |
| scout-4 | **Quality + Operations** (debug tooling, standards, warnings, scripts) | 08_DEBUG, 09_STANDARDS, 10_WARNINGS | Quality + operations. ~14k lines combined. |

#### 1-1. Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-scout-{timestamp}", "description": "Parallel codebase exploration for skeleton" })

// Phase A: Parallel scouts (4 scouts, load-balanced)
TaskCreate({ "subject": "Explore business logic domain", "description": "Explore services, core modules, workflows. Write .scout-report-1.md", "activeForm": "Exploring business logic" })
TaskCreate({ "subject": "Explore data + API domain", "description": "Explore models, schemas, routes, endpoints. Write .scout-report-2.md", "activeForm": "Exploring data + API" })
TaskCreate({ "subject": "Explore infrastructure domain", "description": "Explore env, deps, architecture, structure, config, TODOs. Write .scout-report-3.md", "activeForm": "Exploring infrastructure" })
TaskCreate({ "subject": "Explore quality + operations domain", "description": "Explore debug tooling, standards, warnings, scripts. Write .scout-report-4.md", "activeForm": "Exploring quality + operations" })

// Phase B: Synthesizer (blocked by all scouts)
TaskCreate({ "subject": "Synthesize scout reports into skeleton", "description": "Merge 4 scout reports → generate ai-docs/.skeleton.md", "activeForm": "Synthesizing skeleton" })

TaskUpdate({ "taskId": "5", "addBlockedBy": ["1", "2", "3", "4"] })
```

#### 1-2. Assign & Spawn Scouts (parallel)

```json
TaskUpdate({ "taskId": "1", "owner": "scout-1" })
TaskUpdate({ "taskId": "2", "owner": "scout-2" })
TaskUpdate({ "taskId": "3", "owner": "scout-3" })
TaskUpdate({ "taskId": "4", "owner": "scout-4" })
TaskUpdate({ "taskId": "5", "owner": "synthesizer" })
```

Spawn all 4 scouts in a single message (parallel execution):

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-scout-{timestamp}",
  "name": "scout-1",
  "description": "Explore business logic domain",
  "prompt": "<Scout Prompt — assigned: business logic (07)>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-scout-{timestamp}",
  "name": "scout-2",
  "description": "Explore data + API domain",
  "prompt": "<Scout Prompt — assigned: data + API (05, 06)>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-scout-{timestamp}",
  "name": "scout-3",
  "description": "Explore infrastructure domain",
  "prompt": "<Scout Prompt — assigned: infra (01-04, 11)>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-scout-{timestamp}",
  "name": "scout-4",
  "description": "Explore quality + operations domain",
  "prompt": "<Scout Prompt — assigned: quality + ops (08-10)>"
})
```

#### Scout Prompt Template

```
You are a scout agent in the "ai-docs-scout" team.

Your job is to deeply explore a specific domain of the codebase and produce
a structured exploration report. Another agent (synthesizer) will merge all
scout reports into the final skeleton file.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules) and §1-2 (Phase 0 Skeleton structure)

## Step 2: Your Assigned Domain
[DOMAIN DESCRIPTION — e.g.:]
- scout-1: Business logic — services, core modules, workflows (target: 07_BUSINESS_LOGIC)
- scout-2: Data + API — models, schemas, routes, endpoints (target: 05_DATA_MODELS, 06_API)
- scout-3: Infrastructure — env, deps, architecture, structure, config (target: 01-04, 11_TODO)
- scout-4: Quality + operations — debug tooling, standards, warnings, scripts (target: 08-10)

## Step 3: Explore Your Domain
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Systematically explore source files in your domain:
   - Directory structure (Glob sweep of relevant directories)
   - File purposes and relationships
   - Classification facts: any per-file property that multiple documents might describe independently.
     Grep/read every source file for file-level directives, configuration exports, and module markers.
     Examples by ecosystem (adapt to the actual project):
       * JS/TS: "use client" / "use server", export const dynamic/revalidate/runtime, barrel re-exports
       * Rust: #![no_std], #[cfg(feature)], pub mod boundaries
       * Python: __all__, TYPE_CHECKING blocks, abstract base classes
       * Go: //go:build tags, internal/ package boundaries
   - **Large file detection**: Flag any source file OR ai-docs document exceeding 500 lines.
     These are refactoring candidates — record file path, line count, and a brief reason
     why splitting might help (e.g., "3 unrelated services in one file", "mixing config and logic").
   - **Workload measurement**: Count relevant files, approximate LOC, and key units for your domain.
     Key units are ecosystem-specific: endpoints, tables, migrations, env vars, commands, services,
     scheduled jobs, tests, CLI commands, UI route groups, external integrations, etc.
   - Explicit TODOs/FIXMEs (Grep sweep for TODO, FIXME, HACK, BUG, WORKAROUND)
   - Implicit issues discovered during analysis

   Domain-specific exploration:
   - scout-1 (business logic): service layer, business rules, workflows, state machines, scheduling
   - scout-2 (data + API): DB schema, ORM models, API routes, request/response shapes, auth middleware
   - scout-3 (infrastructure): project identity, tech stack, package manifests, config files, entry points, CI/CD, env templates, directory layout
   - scout-4 (quality + ops): logging, monitoring, debug tools, coding standards/patterns, known pitfalls, operational scripts

## Step 4: Write Scout Report
Write your findings to: ai-docs/.scout-report-N.md (N = your scout number: 1, 2, 3, or 4)

Report format:
```markdown
# Scout Report N — [Domain Name]
> Scout: scout-N

## Value Facts
| Fact | Value | Source File |
|------|-------|------------|
(versions, paths, ports, counts — scalar values relevant to your domain)

## Classification Map
| File | Classification | Evidence |
|------|---------------|----------|
(file → category/directive/flag for every file with a classifiable property)

## Cross-References
| Source (this domain) | Target (other domain) | Relationship |
|---------------------|----------------------|-------------|
(connections you found to other scouts' domains)

## TODO Registry Items
| ID Suggestion | File | Description | Priority |
|--------------|------|-------------|----------|
(TODO, FIXME, HACK, BUG, WORKAROUND found in your domain)

## Large Files (Refactoring Candidates)
| File | Lines | Reason |
|------|-------|--------|
(Any source file or ai-docs document exceeding 500 lines. Include line count and brief split rationale.)

## Workload Signals
| Target Doc | Source Files | Source LOC | Key Units | Risk Notes | Suggested Handling |
|------------|--------------|------------|-----------|------------|--------------------|
(Example: 06_API | 42 | 8900 | endpoints=118, auth layers=3 | high: auth + external webhooks | split by endpoint group + owner)

## File-to-Document Map
| Source Path | Primary Document | Secondary |
|------------|-----------------|-----------|
(key source files/dirs → which documents should reference them)

## Section Outline
Proposed sections for your target document(s), following SPEC §5 requirements:
(e.g., for 07: §7-1 Service Overview, §7-2 Core Workflows, ...)

## Key Observations
- [Notable architectural decisions, patterns, or concerns in your domain]
```

## Step 5: Report Completion
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Scout complete for [domain].\n\nFiles explored: [N]\nValue facts: [N]\nClassifications: [N]\nTODOs found: [N]\nKey observations: [summary]",
  "summary": "Scout [domain] complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### 1-3. After Scouts Complete → Spawn Synthesizer

When all 4 scout tasks are `completed`:

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-scout-{timestamp}",
  "name": "synthesizer",
  "description": "Merge scout reports into skeleton",
  "prompt": "<Synthesizer Prompt>"
})
```

#### Synthesizer Prompt

```
You are the synthesizer agent in the "ai-docs-scout" team.

Your job is to merge the 4 scout reports into a single, coherent skeleton file
that will guide parallel document writers.

## Step 1: Read These Files
1. ai-docs/.scout-report-1.md
2. ai-docs/.scout-report-2.md
3. ai-docs/.scout-report-3.md
4. ai-docs/.scout-report-4.md
5. ai-docs/SPEC.md — §0-2 (for skeleton structure rules, including Workload Matrix and Agent Assignment Plan)

## Step 2: Generate Skeleton
Write to: ai-docs/.skeleton.md

Merge all scout reports into the skeleton per SPEC §1-2:

A. **Shared Facts** (two sub-tables required):
   - **Value Facts**: Merge from all 4 reports. Deduplicate. Resolve conflicts by checking source files.
   - **Classification Map**: Merge from all 4 reports. If two scouts classify the same file differently, check the actual source file and use the correct value.

B. **Cross-Reference Map**: Merge cross-references from all reports. Assign canonical vs mention roles per SPEC rules.

C. **TODO Registry**: Merge TODO items from all reports. Assign final IDs (format per SPEC). Deduplicate items found by multiple scouts.
   - **Large Files**: Merge "Large Files (Refactoring Candidates)" from all reports into the TODO Registry as `implicit-large-*` entries (Priority P2). These will surface in 11_TODO §4 Refactoring Candidates.

D. **Section Numbering Scheme**: Merge section outlines from all reports into a unified numbering scheme for all 11 documents. Ensure no § number collisions.

E. **File-to-Document Map**: Merge from all reports. Resolve any file claimed by multiple documents (assign primary vs secondary).

F. **Workload Matrix**: Merge "Workload Signals" from all reports into a single per-document matrix. Count relative source volume and risk. This is the basis for writer/reviewer sizing.

G. **Agent Assignment Plan**: Create explicit Write Team and Review Team tables from the Workload Matrix. This plan replaces the old fixed writer-1/writer-2/writer-3 partition.
   - Bundle only light related docs.
   - Give medium docs a dedicated writer when practical.
   - Split heavy/critical docs into fragments and assign one doc owner/integrator.
   - Assign reviewers by risk and coverage need, not by writer grouping.
   - Ensure the cross-checker is blocked by all reviewers.

## Step 3: Resolve Conflicts
If scout reports contradict each other (e.g., different classification for the same file, different version numbers):
- Check the actual source file to determine the correct answer
- Apply the correction to the skeleton
- Note resolutions in the skeleton as comments

## Step 4: Quality Check
Before finalizing, verify:
- Every document (01-11) has at least one section in the Section Numbering Scheme
- Every document (01-11) has exactly one final writer/owner in the Agent Assignment Plan
- Heavy/critical documents have split handling or an explicit rationale for a single dedicated writer
- Review Team assignments cover all heavy/critical Workload Matrix key units
- Classification Map has no duplicates or contradictions
- Cross-Reference Map is symmetric (if A→B canonical, B→A is mention)
- Skeleton remains a single file. If approaching 260 lines, compress (shorter descriptions, merge similar items)

## Step 5: Report Completion
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Skeleton synthesized.\n\nValue facts: [N]\nClassifications: [N]\nCross-refs: [N]\nTODO items: [N]\nWorkload: [light/medium/heavy/critical counts]\nWrite agents planned: [N]\nReview agents planned: [N]\nConflicts resolved: [N or 'none']",
  "summary": "Skeleton synthesis complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

#### 1-4. Shutdown Scout Team + Cleanup

After synthesizer reports:

```json
SendMessage({ "type": "shutdown_request", "recipient": "scout-1", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "scout-2", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "scout-3", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "scout-4", "content": "All tasks complete" })
SendMessage({ "type": "shutdown_request", "recipient": "synthesizer", "content": "All tasks complete" })
TeamDelete()
```

**Delete temporary scout reports:**
- `ai-docs/.scout-report-1.md`
- `ai-docs/.scout-report-2.md`
- `ai-docs/.scout-report-3.md`
- `ai-docs/.scout-report-4.md`

**Orchestrator validation** (after cleanup):
1. Confirm `ai-docs/.skeleton.md` exists
2. Validate skeleton structure (read ONLY section headers, not body):
   - `## A. Shared Facts` exists (with both `### Value Facts` and `### Classification Map` sub-headers)
   - `## B. Cross-Reference Map` exists
   - `## C. TODO Registry` exists
   - `## D. Section Numbering Scheme` exists
   - `## E. File-to-Document Map` exists
   - `## F. Workload Matrix` exists
   - `## G. Agent Assignment Plan` exists
   If any section is missing, flag it and consider re-running the synthesizer.
3. Do NOT read skeleton body content — keep orchestrator context lightweight.

---

## Step 2: Write Team (ai-docs-gen, Workload-Based)

### 2-1. Read Skeleton Assignment Plan

Before creating writer tasks, the orchestrator reads only these sections of `ai-docs/.skeleton.md`:

- `## F. Workload Matrix`
- `## G. Agent Assignment Plan`

Do not read the whole skeleton body into the main session. The writer agents read it from disk.

The 12 documents are the output contract. They are not the agent partition. Light docs may be bundled; heavy/critical docs may be split into fragments with a doc owner/integrator.

### 2-2. Validate Assignment Plan

Before spawning writers, apply these checks:

- Every final document `01_ENVIRONMENT.md` through `11_TODO.md` has exactly one final writer/owner responsible for producing the final file.
- Every fragment output under `ai-docs/.fragments/` has a corresponding doc owner/integrator.
- `05_DATA_MODELS.md`, `06_API.md`, and `07_BUSINESS_LOGIC.md` are not bundled together unless the Workload Matrix marks them `light`.
- Any document marked `heavy` or `critical` has either:
  - one dedicated single-doc writer and one targeted reviewer, or
  - multiple fragment writers plus one doc owner/integrator.
- Agent count is plausible:
  - small repo: 2-3 writers
  - medium repo: 3-5 writers
  - large/high-risk repo: 6-10 writer/owner agents

If the plan fails these checks, spawn a short "assignment-fixer" agent to edit only `## G. Agent Assignment Plan`, then re-check. Do not proceed with a broken plan.

### 2-3. Create Team & Tasks Dynamically

Generate one task per row in the skeleton's Write Team plan.

```json
// Generate unique team name to prevent collision with stale teams from crashed runs
// Format: ai-docs-gen-{YYYYMMDD-HHmmss} (e.g., ai-docs-gen-20260220-143022)
TeamCreate({ "team_name": "ai-docs-gen-{timestamp}", "description": "AI docs workload-based parallel writing" })

TaskCreate({ "subject": "writer-env-deps", "description": "Write 01_ENVIRONMENT.md and 02_DEPENDENCIES.md per skeleton assignment", "activeForm": "Writing env/deps docs" })
TaskCreate({ "subject": "writer-api-auth", "description": "Write ai-docs/.fragments/06-api-auth.md per skeleton assignment", "activeForm": "Writing API auth fragment" })
TaskCreate({ "subject": "owner-api", "description": "Integrate 06_API.md from API fragments", "activeForm": "Integrating 06_API.md" })
```

The example above is illustrative only. Use the actual `## G. Agent Assignment Plan` rows.

### 2-4. Assign Dependencies

Use the `Blocked By` column in the Write Team plan:

- Fragment writers are blocked only by the skeleton.
- Bundled/single-doc writers are blocked only by the skeleton.
- Doc owner/integrator tasks are blocked by all fragment writers for that document.
- A doc owner may do source spot checks, but must not redo all fragment work unless the fragment is clearly wrong.

```json
TaskUpdate({ "taskId": "<owner-api-task>", "addBlockedBy": ["<writer-api-auth-task>", "<writer-api-bots-task>"] })
TaskUpdate({ "taskId": "<task-id>", "owner": "<agent-name-from-plan>" })
```

### 2-5. Spawn Writers (parallel, by dependency wave)

Spawn all unblocked writer agents in one message. After a wave completes, spawn newly unblocked doc owners/integrators.

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "writer-env-deps",
  "description": "Write 01_ENVIRONMENT.md and 02_DEPENDENCIES.md",
  "prompt": "<Writer Prompt — assigned from skeleton plan>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "writer-api-auth",
  "description": "Write 06_API auth fragment",
  "prompt": "<Writer Prompt — assigned from skeleton plan>"
})
```

After all fragments for a split doc complete:

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "owner-api",
  "description": "Integrate 06_API.md from fragments",
  "prompt": "<Doc Owner Prompt — assigned from skeleton plan>"
})
```

### Writer Prompt Template

```
You are a documentation writer agent in the "ai-docs-gen" team.

## Step 1: Read These Files
1. ai-docs/SPEC.md — §0 (Hard Rules), §4 (Common Principles), §5 (your assigned docs only)
2. ai-docs/.skeleton.md — Shared Facts, Cross-Reference Map, TODO Registry, Section Numbering, Workload Matrix, Agent Assignment Plan

## Step 2: Your Assigned Outputs
[COPY THE EXACT ROW FROM ## G. Agent Assignment Plan]

You may be one of:
- bundled writer: produce one or more final ai-docs/XX_FILE.md files
- single-doc writer: produce one final ai-docs/XX_FILE.md file
- section/domain fragment writer: produce only ai-docs/.fragments/<doc-section>.md
- doc owner/integrator: merge fragments into one final ai-docs/XX_FILE.md

## Step 3: For Each Assigned Output
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read relevant source files from your assigned `Source Scope`; use the File-to-Document Map and Workload Matrix to avoid missing heavy areas.
3. Write your assigned final file or fragment following:
   - SPEC rules (evidence-based, tables over prose)
   - Skeleton's Shared Facts — BOTH Value Facts AND Classification Map (do NOT re-derive independently)
   - Skeleton's Section Numbering (§ numbers exactly as assigned)
   - Skeleton's Cross-Reference Map (canonical = full detail, mention = one-line + → ref)
   - ## Evidence at end
   - Metadata: > Generated: YYYY-MM-DD | Based on commit: <hash from skeleton>
   - For heavy/critical docs, cover the Workload Matrix key units assigned to you (endpoints, tables, jobs, services, etc.).
4. TaskUpdate({ "taskId": "<id>", "status": "completed" })

## Fragment Rule
If you are a fragment writer:
- Do NOT create or edit the final `ai-docs/XX_FILE.md`.
- Write only your fragment under `ai-docs/.fragments/`.
- Include a local `## Evidence` section for the fragment.
- Include a short "handoff notes" block for the doc owner: coverage completed, possible gaps, and files not inspected.

## Doc Owner / Integrator Rule
If you are a doc owner/integrator:
- Read all fragments for your document.
- Spot-check the highest-risk source files yourself.
- Merge into the final `ai-docs/XX_FILE.md`.
- Remove duplication and resolve contradictory fragment claims by checking source files.
- Ensure the final document follows SPEC section numbering and has one final `## Evidence` section.

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

### 2-6. Monitor & Shutdown Write Team

```json
TaskList()  // check progress periodically
```

**Timeout & retry policy:**
- Check progress every 2-3 minutes via TaskList()
- If an agent's task stays `in_progress` for >10 minutes with no file output, send a status check message
- If still stuck after another 5 minutes, note the failure and proceed with remaining tasks
- After all other tasks complete, attempt the failed task by spawning a replacement agent
- If replacement also fails, the orchestrator writes the document directly as fallback

When all Write Team tasks from the assignment plan are completed:

```json
SendMessage({ "type": "shutdown_request", "recipient": "<each writer/owner agent from plan>", "content": "All tasks complete" })

// After all confirm shutdown
// Uses current team context — no need to specify name
TeamDelete()
```

---

## Step 3: Review Team (ai-docs-review, Workload-Based)

### 3-1. Read Review Assignment Plan

Before creating review tasks, the orchestrator reads the `### Review Team` table under `ai-docs/.skeleton.md` section `## G. Agent Assignment Plan`.

Reviewers are assigned by risk and coverage need, not by writer grouping. A large `06_API.md` may need endpoint-contract and auth reviewers; a large `05_DATA_MODELS.md` may need final-state migration review; a large `07_BUSINESS_LOGIC.md` may need runtime-flow, scheduler, and external-side-effect reviewers.

### 3-2. Create Team & Tasks Dynamically

Create one task per row in the skeleton's Review Team plan, including the cross-checker row.

```json
// Generate unique team name to prevent collision with stale teams from crashed runs
// Format: ai-docs-review-{YYYYMMDD-HHmmss} (e.g., ai-docs-review-20260220-144500)
TeamCreate({ "team_name": "ai-docs-review-{timestamp}", "description": "AI docs workload-based review and verification" })

TaskCreate({ "subject": "reviewer-api-contracts", "description": "Review 06_API endpoint contracts against routes/schemas/auth", "activeForm": "Reviewing API contracts" })
TaskCreate({ "subject": "reviewer-data-final-state", "description": "Review 05_DATA_MODELS against models and migrations final state", "activeForm": "Reviewing data model final state" })
TaskCreate({ "subject": "Verify consistency + write INDEX", "description": "Check §/cross-ref/shared-facts across all docs, write 00_INDEX.md", "activeForm": "Verifying consistency" })
```

The example above is illustrative only. Use the actual review plan rows.

### 3-3. Assign Dependencies

Use the `Blocked By` column in the Review Team plan:

- Content reviewers are blocked by the relevant final document owner/writer tasks.
- The cross-checker is blocked by all reviewers.
- If a heavy/critical doc has multiple reviewers, they may run in parallel against the same final doc, but each must keep edits scoped to their review domain. If two reviewers need to edit the same paragraph, the later reviewer must re-read the current file before editing.

### 3-4. Spawn Reviewers (parallel, by dependency wave)

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-api-contracts",
  "description": "Review and fix 06_API endpoint contracts",
  "prompt": "<Reviewer Prompt — assigned from skeleton plan>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-data-final-state",
  "description": "Review and fix 05_DATA_MODELS final state",
  "prompt": "<Reviewer Prompt — assigned from skeleton plan>"
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
2. ai-docs/.skeleton.md — Shared Facts, Cross-Reference Map, TODO Registry, Section Numbering, Workload Matrix, Agent Assignment Plan

## Step 2: Your Assigned Review Scope
[COPY THE EXACT ROW FROM ## G. Agent Assignment Plan / Review Team]

Review only the assigned documents and source scope. Do not redo another reviewer's domain unless you find a direct contradiction.

## Step 3: For Each Document
1. TaskUpdate({ "taskId": "<id>", "status": "in_progress" })
2. Read the existing document thoroughly
3. Read the ACTUAL SOURCE FILES referenced in the document's ## Evidence section
   — AND search for additional source files the writer may have missed
4. Check against SPEC §5 requirements for this document type:
   - Are all mandatory sections present?
   - Are all items covered? (e.g., every endpoint in 06, every table in 05, every env var in 01)
   - For heavy/critical docs, does coverage match the Workload Matrix key units?
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
  "summary": "Assigned review scope complete"
})

## Working Directory
[ABSOLUTE PATH TO PROJECT ROOT]
```

### 3-5. After Reviewers Complete → Spawn Cross-Checker

When all reviewer tasks from the skeleton's Review Team plan are `completed`:

```json
Agent({
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

**Cross-agent consistency check**: Writers and reviewers are workload-based, so related facts may be touched by different agents.
Contradictions across final documents or fragments (e.g., 03_ARCHITECTURE vs 05_DATA_MODELS describing the same flow differently)
may not be caught by targeted reviewers. You MUST:
- Scan for overlapping topics across document boundaries
- Verify that cross-references between documents resolve correctly
- Fix any factual contradictions between documents by checking source files

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
- The cross-checker only spawns after all reviewer tasks from the assignment plan are `completed`

### 3-6. Shutdown Review Team

After cross-checker reports:

```json
SendMessage({ "type": "shutdown_request", "recipient": "<each reviewer agent from plan>", "content": "All tasks complete" })
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
