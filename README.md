# ai-docs

AI-optimized project documentation generator for Claude Code.

Creates structured docs that enable **any AI** to immediately work on a project — using agent teams for parallel generation and review.

## Architecture

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
"Generate AI docs for this project"
"Create AI documentation"
"Document this project for AI"
```

## How It Works

### Phase 0: Scout

A scout agent explores the entire codebase and produces a **skeleton** — a ~200-line metadata file containing shared facts, cross-reference map, TODO registry, and section numbering. This ensures all writer agents share the same understanding.

### Phase 1: Write Team

3-4 writer agents work in parallel, each reading SPEC.md + skeleton from disk. Each agent reads the relevant source code and writes 2-4 documentation files.

### Phase 2: Review Team

3-4 reviewer agents deeply review the drafts (expected ~65-80% completeness), re-reading source code to find gaps and inaccuracies. After all reviewers finish, a cross-checker verifies structural integrity and writes `00_INDEX.md`.

### Key Design Decisions

| Decision | Reason |
|----------|--------|
| Main session reads no source code | Prevents context overflow in orchestrator |
| Info flows through files, not prompts | SPEC.md + .skeleton.md are read by agents directly |
| Separate write and review teams | Writers have blind spots; fresh eyes catch more |
| Cross-checker runs last | Needs final state of all documents |
| Skeleton limits ~200 lines | Keeps per-agent context overhead minimal |

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Claude Code skill — orchestration logic, agent prompts, tool API reference |
| `SPEC.md` | Generation specification — hard rules, document requirements, verification checklist |

## License

MIT
