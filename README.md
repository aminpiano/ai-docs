# ai-docs

AI-optimized project documentation generator for Claude Code.

Creates structured docs that enable **any AI** to immediately work on a project — using agent teams for parallel generation and review.

## Architecture

Two modes — automatically selected based on whether existing docs are present.

### Full Generation (new projects)

```
Main Session (Orchestrator)
  │
  ├─ Scout agent ──→ .skeleton.md (project analysis)
  │
  ├─ Write Team (parallel)
  │    ├─ writer-1 ──→ docs 01-04
  │    ├─ writer-2 ──→ docs 05-07
  │    └─ writer-3 ──→ docs 08-11
  │
  └─ Review Team (parallel → sequential)
       ├─ reviewer-1 ──→ deep review 01-04
       ├─ reviewer-2 ──→ deep review 05-07
       ├─ reviewer-3 ──→ deep review 08-11
       └─ cross-checker ──→ verify + 00_INDEX.md
```

### Update Mode (existing docs)

```
Main Session (Orchestrator)
  │
  ├─ Audit Team (parallel → sequential)
  │    ├─ auditor-1 ──→ audit 07 (business logic, solo)
  │    ├─ auditor-2 ──→ audit 05-06 (data + API)
  │    ├─ auditor-3 ──→ audit 01-04, 11 (infra + todo)
  │    ├─ auditor-4 ──→ audit 08-10 (quality + ops)
  │    └─ synthesizer ──→ skeleton update + manifest + refactoring list
  │    → User reviews refactoring list
  │
  ├─ Update Write Team (1-3 writers, scaled by affected docs)
  │    └─ Edit only affected documents
  │
  └─ Update Review Team
       ├─ reviewer ──→ deep review modified docs
       └─ cross-checker ──→ cross-ref consistency + INDEX update
```

## Output

```
ai-docs/
├── SPEC.md              # Generation specification (reusable)
├── .skeleton.md         # Project analysis metadata
├── 00_INDEX.md          # Entry point - AI reads this first
├── 01_ENVIRONMENT.md    # Runtime, env vars, commands
├── 02_DEPENDENCIES.md   # Full dependency spec
├── 03_ARCHITECTURE.md   # System structure, data flow (Mermaid)
├── 04_STRUCTURE.md      # Directory tree, entry points
├── 05_DATA_MODELS.md    # DB schema, type definitions
├── 06_API.md            # HTTP endpoints, auth
├── 07_BUSINESS_LOGIC.md # Core logic, server mutations
├── 08_DEBUG.md          # Logs, tests, troubleshooting
├── 09_STANDARDS.md      # Coding conventions, anti-patterns
├── 10_WARNINGS.md       # Side-effects, do-not-touch areas
└── 11_TODO.md           # Bugs, incomplete features, future plans
```

## Installation

### As a Claude Code skill (recommended)

```bash
# macOS/Linux
mkdir -p ~/.claude/skills
cp SKILL.md ~/.claude/skills/ai-project-docs.skill

# Or package as ZIP first
cd /path/to/ai-docs
zip ai-project-docs.skill SKILL.md
cp ai-project-docs.skill ~/.claude/skills/
```

### SPEC.md (optional, per-project customization)

Copy `SPEC.md` into your project's `ai-docs/` directory to customize generation rules. If not present, the skill generates a default SPEC automatically.

```bash
mkdir -p your-project/ai-docs
cp SPEC.md your-project/ai-docs/SPEC.md
```

## Usage

In Claude Code:

```
/ai-project-docs
```

Or natural language:

```
# Full generation (new project)
"Generate AI docs for this project"
"Create AI documentation"
"Document this project for AI"

# Update existing docs
"Update AI docs to reflect recent changes"
"Sync AI docs with current code"
"Check which docs need updating"
```

When existing docs are detected, the skill asks: **"Full regeneration or Update?"**

## How It Works

### Phase 0: Scout

A scout agent explores the entire codebase and produces a **skeleton** — a ~200-line metadata file containing shared facts, cross-reference map, TODO registry, and section numbering. This ensures all writer agents share the same understanding.

### Phase 1: Write Team

3-4 writer agents work in parallel, each reading SPEC.md + skeleton from disk. Each agent reads the relevant source code and writes 2-4 documentation files.

### Phase 2: Review Team

3-4 reviewer agents deeply review the drafts (expected ~65-80% completeness), re-reading source code to find gaps and inaccuracies. After all reviewers finish, a cross-checker verifies structural integrity and writes `00_INDEX.md`.

### Update Mode

When existing docs are present and user selects "Update":

1. **Audit Team** (4 auditors + 1 synthesizer) compares every existing document against actual source code. Identifies both code-driven updates (from git diff) and document-internal issues (stale info, broken cross-refs, SPEC violations). Produces a **refactoring list** for user review.
2. **Update Write Team** (1-3 writers, scaled by affected doc count) surgically edits only the affected documents using Edit tool — no from-scratch rewriting.
3. **Update Review Team** (1 reviewer + 1 cross-checker) verifies modified docs and checks cross-reference consistency between updated and unchanged documents.

### Key Design Decisions

| Decision | Reason |
|----------|--------|
| Main session reads no source code | Prevents context overflow in orchestrator |
| Info flows through files, not prompts | SPEC.md + .skeleton.md are read by agents directly |
| Separate write and review teams | Writers have blind spots; fresh eyes catch more |
| Cross-checker runs last | Needs final state of all documents |
| Skeleton limits ~200 lines | Keeps per-agent context overhead minimal |
| Audit Team (4 auditors, load-balanced) over single scout | Skeleton quality determines overall doc quality — single agent misses too much on large codebases |
| Refactoring list shown to user | User controls scope before writers begin |
| Update uses Edit, not Write | Preserves unchanged content, minimizes diff noise |

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Claude Code skill — orchestration logic, agent prompts, tool API reference |
| `SPEC.md` | Generation specification — hard rules, document requirements, verification checklist |

## License

MIT
