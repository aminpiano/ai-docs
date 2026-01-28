# ai-docs

AI-optimized project documentation generator for Claude Code.

Creates structured docs that enable **any AI** (Claude, GPT, Gemini) to immediately work on a project.

## What it does

Generates a `.ai-docs/` folder with:

```
.ai-docs/
├── INDEX.md        # Entry point - AI reads this first
├── architecture.md # Tech stack, patterns, components
├── api.md          # Endpoints, schemas, auth
├── database.md     # Tables, relationships, queries
├── deployment.md   # Build, deploy, env vars
└── workflows.md    # User flows, state transitions
```

## Installation

Copy `SKILL.md` to your Claude Code skills folder:

```bash
# Windows
cp SKILL.md %USERPROFILE%\.claude\skills\ai-project-docs\SKILL.md

# macOS/Linux
cp SKILL.md ~/.claude/skills/ai-project-docs/SKILL.md
```

## Usage

In Claude Code:

```
"Generate AI docs for this project"
"Create AI documentation"
"Make this project AI-friendly"
```

## Features

### Parallel Task Architecture

For large projects, uses parallel Tasks to prevent context overflow:

| Project Size | Strategy |
|--------------|----------|
| Small (< 50 files) | Single Task |
| Medium (50-200) | 2-3 parallel Tasks |
| Large (200-500) | 5-6 parallel Tasks |
| Very Large (500+) | 10+ Tasks + hierarchical INDEX |

### Minimal Context Usage

Main session reads only ~60 lines total:
- Phase 1: package.json + structure (~50 lines)
- Phase 2-3: Tasks handle everything (0 lines)
- Phase 4: File verification (~10 lines)

### Smart Model Selection

| Task | Model |
|------|-------|
| File scanning | haiku |
| Code extraction | haiku |
| Doc writing | sonnet |
| INDEX generation | sonnet |

## Output Example

```markdown
# MyApp - AI Documentation

> E-commerce platform with React + Node.js

## Quick Reference

| Aspect | Value |
|--------|-------|
| Frontend | React 18, TypeScript |
| Backend | Node.js 20, Express |
| Database | PostgreSQL 15 |

## Docs

| Doc | When to Read |
|-----|--------------|
| [architecture.md](architecture.md) | Tech stack, patterns |
| [api.md](api.md) | API endpoints |
| [database.md](database.md) | Schema, queries |
```

## License

MIT
