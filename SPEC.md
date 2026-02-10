# AI-Docs Generation Specification

> Universal specification for generating `/ai-docs/` project documentation (12 files) using an AI agent team.
> Not tied to any specific project. Applicable to any repository.

---

## §0 Hard Rules

### §0-1 Evidence-Based Writing

- Write only based on source code, config, and package files that actually exist in the repository.
- Speculation, inference, general knowledge, and web search are prohibited.
- Items that cannot be confirmed from the project:
  - Identity fields (name, purpose, status, stack, etc.) → mark as `UNKNOWN`
  - Not applicable (e.g., no DB) → mark as `N/A`
- Do not delete sections; keep them with the appropriate marker.

**Boundary between evidence-based statements and speculation:**

| Allowed (observable facts) | Prohibited (intent/motive speculation) |
|---------------------------|---------------------------------------|
| "Field X is String type; no numeric validation exists in the API, so non-numeric input is stored" | "Field X is String to allow range inputs" |
| "Schema default is `A` but the API inserts `B`" | "The developer left `A` for future use" |

Principle: **If behavior is observable in code, describe it. If the reason is not in code/comments, do not describe it.**

### §0-2 Evidence Section (Mandatory)

- Every document file must include a `## Evidence` section at the end.
- Evidence lists the exact repository file paths referenced when writing that document.
- Line range notation is not required.

### §0-3 Output Format

- Output directory: `ai-docs/` (relative to project root). Fixed.
- Generate all 12 files listed in §2. No omissions.
- Mermaid must use ` ```mermaid ` fenced code blocks.
- Commands, config, and examples must be in code blocks.
- Tone: Dry / Technical / Direct / Imperative. No hedging or speculative language.
- Encoding: UTF-8. Verify non-ASCII characters (CJK, box-drawing `├└─`, etc.) are intact.

### §0-4 Document Metadata

Top of every document file, directly below the title:

```markdown
> Generated: YYYY-MM-DD | Based on commit: <short hash or "uncommitted">
```

- `Generated`: date of document creation.
- `Based on commit`: result of `git rev-parse --short HEAD`. If no git or no commits, use `uncommitted`.

### §0-5 Cross-Reference Principle

When the same issue/concept spans multiple documents, designate one document as the **canonical source** with full detail. Other documents include a **one-line summary + cross-reference** in the format: `→ Detail: XX_FILE.md §N`.

**Canonical source priority:**

| Issue Type | Canonical Source |
|-----------|-----------------|
| Data structure / schema | 05_DATA_MODELS.md |
| Business logic / server-side mutation | 07_BUSINESS_LOGIC.md |
| System architecture / data flow | 03_ARCHITECTURE.md |
| Risk / side-effects / warnings | 10_WARNINGS.md |
| Error / debugging / troubleshooting | 08_DEBUG.md |
| Coding conventions / standards | 09_STANDARDS.md |
| Dependencies / versions | 02_DEPENDENCIES.md |
| Environment / runtime / commands | 01_ENVIRONMENT.md |

Cross-reference target section numbers use those **pre-assigned in the Phase 0 skeleton** (→ §1-2).

### §0-6 Verification (Self-Check)

The Phase 2 review team checks the following and fixes issues immediately (reviewers handle content, cross-checker handles structure):

1. File completeness — all 12 files exist
2. Required sections — every mandatory section per document is present
3. Evidence presence — every document ends with `## Evidence`
4. Inter-document consistency — especially between 09_STANDARDS.md (current rules) and 11_TODO.md (planned changes); conflicts must note `"Current pattern, change planned (→ 11_TODO #ID)"`
5. Cross-reference validity — every referenced `XX_FILE.md §N` actually exists
6. Shared Facts consistency — all documents agree with the skeleton on shared facts (versions, paths, etc.)
7. Non-ASCII encoding integrity
8. Quick Start executability — logically verify that following the documented command sequence actually starts the project

**Check result output:**
- Format: Markdown checkbox list (`- [x]` / `- [ ]`)
- Location: included in the cross-checker agent's final response

### §0-7 INDEX Writing Order

`00_INDEX.md` is always written **last, in Phase 2**, synthesizing all files.

### §0-8 Security

Never record environment variable values, tokens, or passwords in documents.

### §0-9 Section Numbering

All `##`-level sections in every document receive sequential numbers. Format: `## §1 Section Name`, `## §2 Section Name`, ...

These numbers are cross-reference targets (e.g., `→ Detail: 10_WARNINGS.md §1`).

Numbers are **pre-assigned in the Phase 0 skeleton**. Phase 1 agents must follow them.

---

## §1 Execution Model (Scout → Write Team → Review Team)

### §1-1 Why This Model

| Approach | Problem |
|----------|---------|
| Fully parallel (12 agents at once) | No shared knowledge → cross-reference failure, fact conflicts |
| Fully sequential (1 agent writes 12) | Later docs must read all prior docs → context explosion |
| Write only (no review) | Initial drafts are ~65-80% complete. Gaps, inaccuracies, missing items remain |
| Single verifier for all docs | Medium+ projects: 11 docs exceed single agent context window |
| **Scout → Write Team → Review Team** | Skeleton ensures consistency, parallel writing for speed, parallel review for quality |

**Architecture**: Main session is a lightweight orchestrator. All code reading, writing, and verification happens in spawned agents. Information flows through files (`SPEC.md`, `.skeleton.md`, doc files), not through the main session's context.

```
Main Session (Orchestrator)
  │
  ├─ Phase 0: Scout agent ──→ ai-docs/.skeleton.md
  │
  ├─ Phase 1: Write Team (parallel)
  │    ├─ writer-1 ──→ docs 01-04
  │    ├─ writer-2 ──→ docs 05-07
  │    └─ writer-3 ──→ docs 08-11
  │
  └─ Phase 2: Review Team (parallel → sequential)
       ├─ reviewer-1 ──→ deep review 01-04 ─┐
       ├─ reviewer-2 ──→ deep review 05-07 ─┤ (parallel)
       ├─ reviewer-3 ──→ deep review 08-11 ─┘
       │   (all reviewers done)
       └─ cross-checker ──→ structural verify + 00_INDEX.md (sequential)
```

### §1-2 Phase 0 — Skeleton Generation (Scout Agent)

**Input**: entire repository root

**Output**: `ai-docs/.skeleton.md` (file on disk — agents read it directly, not passed via prompt)

**Skeleton required sections:**

#### A. Shared Facts

Facts used identically across all documents. Prevents inter-document contradictions.

Include (where applicable):
- git commit hash
- Language/runtime version (with derivation source)
- Framework version
- DB type/version
- Package manager type
- Any fact referenced in 2+ documents

Example format:
```
- commit: a1b2c3d
- Runtime: Node.js >=18.17 (source: framework X requires >=18.17)
- DB: PostgreSQL 15 (source: docker-compose.yml)
```

#### B. Cross-Reference Map

For each issue that spans multiple documents, designate canonical source and mention locations.

| Issue ID | Summary | Canonical | Mention |
|---------|---------|-----------|---------|
| (filled after code analysis) | ... | XX_FILE §N | YY §M, ZZ §K |

Issue IDs are free-form but must be unique across all documents.

#### C. TODO Registry (Pre-assigned IDs)

Pre-assign IDs for items in 11_TODO.md. Other documents reference these IDs.

| ID | Summary | Priority |
|----|---------|----------|
| (filled after code analysis) | ... | P0–P3 |

ID rules:
- From explicit comments (`TODO`, `FIXME`, etc.) → `TODO-N` (sequential)
- From code analysis (no comment) → `implicit-{keyword}` (e.g., `implicit-no-auth`)

#### D. Section Numbering Scheme

Pre-assign `§` numbers for each document. Phase 1 agents use these exactly.

Example format:
```
### 01_ENVIRONMENT.md
§1 Runtime
§2 Environment Variables
§3 Command Palette
§4 Not Configured
...

(all 11 documents — excluding 00_INDEX.md)
```

Section composition is determined by project analysis. Include N/A mandatory sections with `(N/A)` marker.

**Skeleton constraints:**
- 200 lines or fewer recommended. Minimize per-agent context overhead.
- No body content. Metadata only.

### §1-3 Phase 1 — Write Team (Parallel, 3-4 Agents)

A dedicated team of writer agents generates 11 documents in parallel.

**Each agent reads from disk (NOT passed via prompt):**
1. `ai-docs/SPEC.md` — §0, §4, §5 (their assigned docs only)
2. `ai-docs/.skeleton.md` — full skeleton
3. Source code/config files needed for their documents (identified from skeleton + own exploration)

**Each agent's output:** 2-4 `/ai-docs/XX_FILE.md` files

Write team is shut down after all documents are generated.

**Rules:**
- `00_INDEX.md` is NOT generated in this phase. (Generated in Phase 2)
- Use Shared Facts from the skeleton for all factual statements. Do not independently analyze versions/paths.
- Follow the Cross-Reference Map for canonical/mention distinction:
  - Your document is canonical for an issue → write full detail
  - Your document is mention for an issue → one-line summary + `→ Detail: XX_FILE.md §N`
- Follow the Section Numbering Scheme for `§` numbers.
- Use TODO Registry IDs for cross-references. (e.g., `→ 11_TODO #implicit-no-auth`)

**11_TODO.md agent additional rules:**
- Include ALL items from the TODO Registry without omission.
- Issues discovered during Phase 1 code analysis get `EXTRA-N` IDs.

**09_STANDARDS.md agent additional rules:**
- Anti-patterns that are in the TODO Registry MUST note: `"Current pattern, change planned (→ 11_TODO #ID)"`.

### §1-4 Phase 2 — Review Team (Parallel Reviewers → Cross-Checker)

A dedicated review team improves document quality and ensures structural integrity.
Initial drafts from Phase 1 are expected to be ~65-80% complete.

#### Phase 2a — Parallel Deep Review (3-4 Reviewer Agents)

Each reviewer agent handles 2-4 documents:

**Each reviewer:**
1. Reads `ai-docs/SPEC.md` and `ai-docs/.skeleton.md` from disk
2. Reads each assigned document
3. Re-reads the ACTUAL source files referenced in the document's Evidence section
4. Searches for additional source files the writer may have missed
5. Checks against SPEC §5 requirements:
   - All mandatory sections present?
   - All items covered? (every endpoint, table, env var, etc.)
   - Descriptions accurate against source code?
   - Cross-references valid per skeleton?
   - § numbers match skeleton?
   - Evidence section complete?
6. Fixes all issues directly (Edit tool):
   - Adds missing content
   - Corrects inaccuracies
   - Expands thin sections
   - Completes evidence lists

**Goal:** bring each document from ~65-80% to ~95%+ completeness.

Review team reviewers are shut down after all document reviews complete.

#### Phase 2b — Cross-Checker (Single Agent, runs AFTER all reviewers)

The cross-checker does NOT deeply read document body content — focuses on structure and metadata:

1. Verifies all §0-6 checklist items across 11 documents:
   - File existence, required sections, Evidence presence
   - Cross-reference validity (`→ Detail: XX_FILE.md §N` points to real sections)
   - Shared Facts consistency with skeleton
   - 09↔11 consistency (anti-patterns reference TODO IDs)
   - Non-ASCII encoding integrity
   - Quick Start logical executability
2. Fixes structural issues directly
3. Writes `00_INDEX.md` last (synthesizing all files per §3)
4. Outputs verification checklist as markdown checkboxes

Review team is shut down after cross-checker completes.

---

## §2 File List (Mandatory)

```
ai-docs/00_INDEX.md        — Project summary + full document map           (Phase 2)
ai-docs/01_ENVIRONMENT.md  — Runtime, env vars, commands                   (Phase 1)
ai-docs/02_DEPENDENCIES.md — Full dependency spec                          (Phase 1)
ai-docs/03_ARCHITECTURE.md — System structure, data flow (Mermaid)         (Phase 1)
ai-docs/04_STRUCTURE.md    — Directory tree, entry points, config files    (Phase 1)
ai-docs/05_DATA_MODELS.md  — DB schema, type definitions                   (Phase 1)
ai-docs/06_API.md          — HTTP endpoints, auth, external APIs           (Phase 1)
ai-docs/07_BUSINESS_LOGIC.md — Core logic, server mutations, edge cases    (Phase 1)
ai-docs/08_DEBUG.md        — Logs, tests, error patterns, troubleshooting  (Phase 1)
ai-docs/09_STANDARDS.md    — Coding conventions, anti-patterns             (Phase 1)
ai-docs/10_WARNINGS.md     — Side-effects, do-not-touch areas              (Phase 1)
ai-docs/11_TODO.md         — Bugs, incomplete features, future plans       (Phase 1)
```

---

## §3 00_INDEX.md Rules

Written last in Phase 2. Must include these 4 sections:

### §3-1 Project Identity

| Field | Rule |
|-------|------|
| Project Name & Purpose | One-sentence definition (use `UNKNOWN` if no evidence) |
| Project Status | `Dev` / `Prod` / `Maintenance` (use `UNKNOWN` if no evidence) |
| Primary Tech Stack | Core technologies only (use `UNKNOWN` if no evidence) |

### §3-2 Quick Start

- Minimum command sequence from install to running.
- Following this alone must start the project locally (within what the repo supports).
- State prerequisites (required runtimes/tools).
- **Do not omit implicit prerequisite steps.** Env file copy/creation, DB init, migrations, key generation — do not assume "they'll know." List every step explicitly.
- If DB connection info is needed, indicate where defaults can be found in repo config files (docker-compose, env templates, etc.). Do NOT record actual credential values (§0-8 security rule).

### §3-3 Document Map

- Each file's path / role / when to reference / recommended read order.
- Must include a Common Task Routing table. Default entries:

| Task | Read Sequence |
|------|--------------|
| DB/storage schema change | 05 → 10 → 03 → 06 |
| Error / debugging | 08 → 07 → 10 |
| New feature | 03 → 07 → 09 → 05 → 10 |
| Add/upgrade dependency | 02 → 10 |
| Add API endpoint | 06 → 03 → 09 → 05 |
| Environment / deployment | 01 → 02 |
| Add/modify tests | 08 → 09 |
| Style / UI change | 09 → 10 → 04 |

- Add project-specific high-frequency tasks if applicable. Remove inapplicable rows.

### §3-4 Critical Warnings Summary

- Summarize `10_WARNINGS.md` highlights in 3–5 lines.
- Only include constraints that must be known before any work.
- **Do not write details.** Each item is a one-line summary + `→ Detail: 10_WARNINGS.md §N`.

---

## §4 Per-Document Writing Principles (Common)

| Principle | Description |
|-----------|-------------|
| Target Audience | AI Agent |
| Format priority | Table / bullet / key-value / code block > prose |
| N/A sections | Mark as `N/A` but keep the section |
| Evidence | `## Evidence` at the end of every document |
| Empty item consolidation | Merge similar empty items (e.g., no CI/CD, no tests) into one table instead of 3 separate sections — save tokens |
| Directory tree | Exclude `node_modules`, build artifacts, caches, virtual envs, language-specific dependency dirs |
| Section numbering | Follow skeleton's Section Numbering Scheme (§0-9) |
| Ownership principle | Each document details only its domain. Other domains → cross-reference per §0-5 |

---

## §5 Per-Document Required Content

### 01_ENVIRONMENT.md

- Runtime version and requirements
- Environment Variables: key list from env files (`.env`, etc.), type, role (NO values)
  - Note conditional requirements. If required status varies by environment (Docker / local / CI), state the condition.
- Command Palette: exact mapping for Build / Run / Test / Lint / Deploy
- Unconfigured items → consolidate into one "Not Configured" table

### 02_DEPENDENCIES.md

- Core dependency list with roles
- devDependencies vs production distinction (where the ecosystem supports it)
- Version compatibility constraints (pinned, paired versions, etc.)
- Inter-dependency relationships
- External system dependencies (OS, services, daemons)
- Install commands and lock file management rules

### 03_ARCHITECTURE.md

- System Architecture: `mermaid graph TD`
- Data Flow: `mermaid sequenceDiagram`
  - Cover all major state transition flows. Include not just happy path but also manual fallbacks, error recovery, and alternative flows.
- Module Dependencies: core module relationships (table or diagram)
- When generalizing, note exceptions explicitly. If writing "all X are Y", note exceptions in parentheses or footnotes.

### 04_STRUCTURE.md — Ownership Restriction Applied

- Directory tree (exclude unnecessary items; include doc structure like `ai-docs/`)
- Entry point files
- Config file locations and roles
- **Do not describe detailed behavior of each file/directory.** File path and one-line role only; detail → cross-reference to the owning document. (e.g., `route.ts — API endpoint → Detail: 06_API.md §N`)

### 05_DATA_MODELS.md

- Express DB/storage schema as type definitions (default: the project's language type notation. TypeScript → interface, Python → dataclass/TypedDict, etc.). If no DB, document data storage structure/format.
- Table/collection relationships, indexes, constraints
- Enum/constant values: describe business meaning. Analyze usage patterns in code to determine what each value means in Description column. Only use `UNKNOWN` when analysis cannot determine meaning.
- Untyped fields (JSON/JSONB, any-casting, dynamic objects, etc.): reverse-engineer the actual shape used in code and document as a separate type definition.

### 06_API.md — Ownership Restriction Applied

**Ownership scope: endpoint contract — URL, method, request/response shape, status codes, error responses.**

- Only cover endpoints callable directly by URL path + HTTP method.
- All endpoints: path, method, parameters, response format
- Document error responses. Specify possible error states (4xx, 5xx) and response body shape.
- Authentication/authorization methods
- External APIs: list, endpoints, auth, key management location/method (NO values)
- **Framework server-side mutation mechanisms** (form actions, server functions, RPC, etc. — things not directly callable by URL) are NOT included here. → 07_BUSINESS_LOGIC.md

**06↔07 boundary rule:**

| 06_API.md covers | 07_BUSINESS_LOGIC.md covers |
|-----------------|---------------------------|
| "This endpoint returns 400 when field X is missing" (what) | "Validation iterates field list checking type and empty values" (how) |
| Request/response **shape** | Request processing **implementation** |
| Status codes and error response format | Validation order, business rules, side-effects |

### 07_BUSINESS_LOGIC.md

- Core logic operation principles
- Complex algorithms summarized as pseudo-code
- Edge cases and exception handling
- Full list of server-side mutation functions (form actions, server functions, RPC, etc.). For each: parameters, return, preconditions, side-effects, errors.
- When referencing data structures, do not omit the full shape. Show the complete structure or cross-reference `05_DATA_MODELS.md §N`.

### 08_DEBUG.md

- Log location, format, level configuration
- Test execution commands, coverage scope
- Frequent error patterns and solutions
  - Record error messages verbatim. Verify non-ASCII character (CJK, etc.) encoding is intact.
- Health check methods
- Troubleshooting flows: step-by-step guide per major failure scenario (`symptom → check steps → resolution`). Cover at minimum:
  1. Application startup failure
  2. DB/storage connection failure (if applicable)
  3. Major external dependency failure (if applicable)
  4. Production environment failure (if production config is found in repo)

### 09_STANDARDS.md

- Linter/formatter config, naming conventions
- Patterns used consistently in the project
- Anti-Patterns: things that must never be done
- **Anti-patterns in the TODO Registry** must note: `"Current pattern, change planned (→ 11_TODO #ID)"`. This reference must be applied to **every applicable item** without exception.

### 10_WARNINGS.md

- Files/modules with high side-effects (tier classification where possible)
- Parts that must never be touched
- Approaches that caused problems in the past (repo evidence only — git history, code comments, docs, etc.)

### 11_TODO.md

**Known Bugs section:**
- Items found from explicit comments (`TODO`, `FIXME`, `HACK`, `BUG`, `WORKAROUND`).
- If none found, mark as `None found`.

**Implicit Issues section:**
- Potential issues found through code analysis (no explicit comments).
- Do not mix the two sections.

**Incomplete Features / Refactoring Candidates / Future Plans** (repo evidence only)

**All items have a Priority column:**

| Priority | Criteria |
|----------|----------|
| P0 | Data loss / security vulnerability / production failure potential |
| P1 | Missing functionality directly impacting usability |
| P2 | Code quality / maintainability / technical debt |
| P3 | Nice-to-have / future expansion |

**P0 internal priority (tiebreaker):**

When multiple P0 items exist, prioritize in this order:
1. Security vulnerabilities (data exposure, auth bypass)
2. Data loss/integrity (schema mismatch, data loss)
3. Availability (DoS, service outage)

**TODO Registry compliance:**
- Include ALL items from Phase 0 skeleton's TODO Registry without omission.
- Items additionally discovered in Phase 1 get `EXTRA-N` IDs.

---

## §6 Document Update Policy

### Regeneration Triggers

Regenerate documents when any of the following occur:
- DB/storage schema change
- API endpoint added/deleted/changed
- Dependency major version change
- Major directory structure change

### Regeneration Method

- **Full regeneration** is the default. (Partial updates can break cross-reference consistency)
- On regeneration, execute this spec's full process (Phase 0 → 1 → 2) from scratch.
- Diff against previous documents to verify no unintended information loss.
