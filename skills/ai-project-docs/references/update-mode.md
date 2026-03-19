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

**Partitioning is load-based, NOT sequential.** Documents vary wildly in evidence file count and source code volume. The default partition below is optimized for a medium-large project (~50k lines). Adjust if your project's document sizes differ significantly.

#### Default Auditor Partition (4 auditors)

| Auditor | Documents | Rationale |
|---------|-----------|-----------|
| auditor-1 | **07_BUSINESS_LOGIC** | Heaviest doc (~45 evidence files, ~14k lines of services). Must be solo. |
| auditor-2 | **05_DATA_MODELS, 06_API** | Data + API layer share schemas. ~11k lines combined. |
| auditor-3 | **01_ENVIRONMENT, 02_DEPENDENCIES, 03_ARCHITECTURE, 04_STRUCTURE, 11_TODO** | Infrastructure + structure + todos. ~10k lines combined. |
| auditor-4 | **08_DEBUG, 09_STANDARDS, 10_WARNINGS** | Quality + operations. ~14k lines combined. |

> **Why not 3?** In a project with 50k+ lines, the business logic doc alone can exceed a single agent's context budget. Cramming 05+06+07 into one auditor (~25k source lines) guarantees degraded output.

#### U-1.1 Create Team & Tasks

```json
TeamCreate({ "team_name": "ai-docs-audit-{timestamp}", "description": "Audit existing docs vs code changes" })

// Phase A: Parallel auditors (4 auditors, load-balanced)
TaskCreate({ "subject": "Audit 07_BUSINESS_LOGIC", "description": "Audit 07_BUSINESS_LOGIC.md against actual code + git diff. Solo — heaviest document.", "activeForm": "Auditing 07_BUSINESS_LOGIC" })
TaskCreate({ "subject": "Audit docs 05-06", "description": "Audit 05_DATA_MODELS, 06_API against actual code + git diff", "activeForm": "Auditing docs 05-06" })
TaskCreate({ "subject": "Audit docs 01-04, 11", "description": "Audit 01_ENVIRONMENT, 02_DEPENDENCIES, 03_ARCHITECTURE, 04_STRUCTURE, 11_TODO against actual code + git diff", "activeForm": "Auditing docs 01-04, 11" })
TaskCreate({ "subject": "Audit docs 08-10", "description": "Audit 08_DEBUG, 09_STANDARDS, 10_WARNINGS against actual code + git diff", "activeForm": "Auditing docs 08-10" })

// Phase B: Synthesizer (blocked by all auditors)
TaskCreate({ "subject": "Synthesize audit reports", "description": "Merge 4 audit reports → update skeleton + generate manifest + refactoring list", "activeForm": "Synthesizing audit results" })

TaskUpdate({ "taskId": "5", "addBlockedBy": ["1", "2", "3", "4"] })
```

#### U-1.2 Assign & Spawn Auditors (parallel)

```json
TaskUpdate({ "taskId": "1", "owner": "auditor-1" })
TaskUpdate({ "taskId": "2", "owner": "auditor-2" })
TaskUpdate({ "taskId": "3", "owner": "auditor-3" })
TaskUpdate({ "taskId": "4", "owner": "auditor-4" })
TaskUpdate({ "taskId": "5", "owner": "synthesizer" })
```

Spawn all 4 auditors in a single message (parallel execution):

```json
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-1",
  "description": "Audit 07_BUSINESS_LOGIC vs actual code",
  "prompt": "<Auditor Prompt — assigned: 07>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-2",
  "description": "Audit docs 05-06 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 05, 06>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-3",
  "description": "Audit docs 01-04, 11 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 01, 02, 03, 04, 11>"
})
Agent({
  "subagent_type": "general-purpose",
  "team_name": "ai-docs-audit-{timestamp}",
  "name": "auditor-4",
  "description": "Audit docs 08-10 vs actual code",
  "prompt": "<Auditor Prompt — assigned: 08, 09, 10>"
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
[LIST — examples by auditor:]
- auditor-1: ai-docs/07_BUSINESS_LOGIC.md (Task ID: 1) — solo, heaviest doc
- auditor-2: ai-docs/05_DATA_MODELS.md, ai-docs/06_API.md (Task ID: 2)
- auditor-3: ai-docs/01_ENVIRONMENT.md, 02, 03, 04, 11_TODO.md (Task ID: 3)
- auditor-4: ai-docs/08_DEBUG.md, 09, 10_WARNINGS.md (Task ID: 4)

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

When all 4 auditor tasks are `completed`:

```json
Agent({
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

Your job is to merge the 4 auditor reports into a coherent update plan:
update the skeleton, and generate the update manifest with a refactoring list.

## Step 1: Read These Files
1. ai-docs/.audit-report-1.md
2. ai-docs/.audit-report-2.md
3. ai-docs/.audit-report-3.md
4. ai-docs/.audit-report-4.md
4. ai-docs/.skeleton.md (current)
5. ai-docs/SPEC.md — §0-2 (for skeleton structure rules)

## Step 2: Update Skeleton (in-place Edit)
Based on the 4 audit reports, update ai-docs/.skeleton.md:
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
(Merged and deduplicated from all 4 audit reports)

### Document-Internal Issues
| Affected Doc | Section | Issue | Priority |
|-------------|---------|-------|----------|
(Merged and deduplicated from all 4 audit reports)

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

## Writer Partition Recommendation

Based on audit findings, recommend how to split work across writers.
Consider: document size (line count), number of update instructions, sub-docs count, and P0 items.

| Writer | Assigned Docs | Est. Edit Count | Rationale |
|--------|--------------|-----------------|-----------|
(e.g., heavy docs like 07 with sub-docs should be solo or paired only with light docs)

Recommended writer count: N
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
SendMessage({ "type": "shutdown_request", "recipient": "auditor-4", "content": "All tasks complete" })
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

Use the **Writer Partition Recommendation** from the manifest (generated by Synthesizer in Step U-1).
The Synthesizer analyzes document sizes, edit counts, sub-doc counts, and P0 items to recommend
an optimal writer count and partition.

**Fallback rule** (if manifest has no recommendation):

| Affected Docs | Writers | Assignment |
|--------------|---------|------------|
| 1–2 | 1 writer | All docs to writer-1 |
| 3–5 | 2 writers | Split evenly by edit count |
| 6+ | 3+ writers | Split by load, not by number range |

**Load-balancing principles:**
- Heavy docs (07 with sub-docs) should be solo or paired only with light docs
- Never assign 07 + sub-docs together with 05 or 06 to the same writer
- Count sub-docs as separate units (07 + 5 sub-docs = 6 units, not 1)
- P0-heavy docs get priority for dedicated writers

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
Agent({
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

Agent({
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
Agent({
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
   - `ai-docs/.audit-report-4.md`
   - `ai-docs/.update-manifest.md`
2. **Report to user**:
   - Documents updated: [list]
   - Documents unchanged: [list]
   - Cross-checker results
   - Any unresolved issues or escalations
