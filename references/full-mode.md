## Step 1: Scout Team → Skeleton File (ai-docs-scout)

Parallel scout team explores the codebase by domain partition, then a synthesizer merges findings into the skeleton. Main session does NOT read source code.

**Partitioning is load-based, NOT sequential.** Documents vary wildly in source code volume. The default partition below is optimized for a medium-large project (~50k lines). Adjust if your project's domain sizes differ significantly.

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
5. ai-docs/SPEC.md — §0-2 (for skeleton structure rules)

## Step 2: Generate Skeleton
Write to: ai-docs/.skeleton.md

Merge all scout reports into the skeleton per SPEC §1-2:

A. **Shared Facts** (two sub-tables required):
   - **Value Facts**: Merge from all 4 reports. Deduplicate. Resolve conflicts by checking source files.
   - **Classification Map**: Merge from all 4 reports. If two scouts classify the same file differently, check the actual source file and use the correct value.

B. **Cross-Reference Map**: Merge cross-references from all reports. Assign canonical vs mention roles per SPEC rules.

C. **TODO Registry**: Merge TODO items from all reports. Assign final IDs (format per SPEC). Deduplicate items found by multiple scouts.

D. **Section Numbering Scheme**: Merge section outlines from all reports into a unified numbering scheme for all 11 documents. Ensure no § number collisions.

E. **File-to-Document Map**: Merge from all reports. Resolve any file claimed by multiple documents (assign primary vs secondary).

## Step 3: Resolve Conflicts
If scout reports contradict each other (e.g., different classification for the same file, different version numbers):
- Check the actual source file to determine the correct answer
- Apply the correction to the skeleton
- Note resolutions in the skeleton as comments

## Step 4: Quality Check
Before finalizing, verify:
- Every document (01-11) has at least one section in the Section Numbering Scheme
- Classification Map has no duplicates or contradictions
- Cross-Reference Map is symmetric (if A→B canonical, B→A is mention)
- Skeleton remains a single file. If approaching 200 lines, compress (shorter descriptions, merge similar items)

## Step 5: Report Completion
TaskUpdate({ "taskId": "<id>", "status": "completed" })

SendMessage({
  "type": "message",
  "recipient": "leader",
  "content": "Skeleton synthesized.\n\nValue facts: [N]\nClassifications: [N]\nCross-refs: [N]\nTODO items: [N]\nConflicts resolved: [N or 'none']",
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
   If any section is missing, flag it and consider re-running the synthesizer.
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
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "doc-writer-1",
  "description": "Write docs 01-04",
  "prompt": "<Writer Prompt — assigned: 01, 02, 03, 04>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-gen",
  "name": "doc-writer-2",
  "description": "Write docs 05-07",
  "prompt": "<Writer Prompt — assigned: 05, 06, 07>"
})
Agent({
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
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-1",
  "description": "Review and fix docs 01-04",
  "prompt": "<Reviewer Prompt — assigned: 01, 02, 03, 04>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-review",
  "name": "reviewer-2",
  "description": "Review and fix docs 05-07",
  "prompt": "<Reviewer Prompt — assigned: 05, 06, 07>"
})
Agent({
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
