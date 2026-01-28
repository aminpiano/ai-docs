---
name: ai-project-docs
description: |
  Generate AI-optimized project documentation for any codebase. Creates structured, token-efficient docs that enable ANY AI (Claude, GPT, Gemini) to immediately work on the project.

  Use when: (1) Setting up a new project for AI collaboration, (2) User asks to "document this project for AI", (3) Creating architecture docs, (4) Making project AI-readable, (5) "Create AI docs", "generate project map", "make this AI-friendly"
---

# AI Project Documentation Generator

Generate comprehensive, AI-optimized documentation that enables any AI to work on a project immediately.

## Output Structure

### Small/Medium Projects (< 300 lines per doc)

```
.ai-docs/
├── INDEX.md          # Entry point - AI reads this first
├── architecture.md   # Tech stack, structure, patterns
├── api.md            # Endpoints, requests, responses
├── database.md       # Schema, relationships, queries
├── deployment.md     # Build, deploy, environments
└── workflows.md      # Key business logic flows
```

### Large Projects (300+ lines per doc)

Split into hierarchical structure:

```
.ai-docs/
├── INDEX.md              # Master index (< 200 lines)
├── architecture/
│   ├── INDEX.md          # Architecture overview + links
│   ├── frontend.md       # Frontend patterns
│   ├── backend.md        # Backend patterns
│   └── infrastructure.md # Infra details
├── api/
│   ├── INDEX.md          # All endpoints table (method, path, summary)
│   ├── auth.md           # Auth endpoints detail
│   ├── users.md          # User endpoints detail
│   └── products.md       # Product endpoints detail
├── database/
│   ├── INDEX.md          # All tables overview
│   ├── schema.md         # Full schema definitions
│   └── queries.md        # Common query patterns
├── deployment/
│   └── INDEX.md          # Usually single file is enough
└── workflows/
    ├── INDEX.md          # Workflow list
    ├── auth-flow.md      # Authentication flow
    └── checkout-flow.md  # Checkout flow
```

### Scaling Rules

| Doc Size | Action |
|----------|--------|
| < 300 lines | Single file |
| 300-600 lines | Split into 2-3 files with INDEX |
| 600+ lines | Split by domain/feature |

**INDEX.md constraints:**
- Master INDEX: < 200 lines (quick overview only)
- Section INDEX: < 150 lines (table of contents + summaries)
- Detail files: < 500 lines each

**Section INDEX pattern:**

```markdown
# API Documentation

## Endpoints Overview

| Method | Path | Summary | Detail |
|--------|------|---------|--------|
| POST | /auth/login | User login | [auth.md](auth.md#login) |
| POST | /auth/register | User register | [auth.md](auth.md#register) |
| GET | /users/:id | Get user | [users.md](users.md#get-user) |
| PUT | /users/:id | Update user | [users.md](users.md#update-user) |
...

## Auth Endpoints → [auth.md](auth.md)
## User Endpoints → [users.md](users.md)
## Product Endpoints → [products.md](products.md)
```

This way AI reads section INDEX first, then only the specific file needed.

## Large Project Strategy

For projects with 50+ files, use parallel Tasks to prevent main session context overflow.

**Critical Rule**: Main session NEVER reads generated docs. All doc generation happens in Tasks.

### Phase Overview

```
Phase 1: Initial Scan (main session)
├── Read package.json, directory structure only (~50 lines)
├── Count files, estimate project size
├── Create .ai-docs/ folder
└── Plan Task distribution

Phase 2: Section Docs (parallel Tasks, haiku/sonnet)
├── Task: api-analyzer      → api.md
├── Task: db-analyzer       → database.md
├── Task: arch-analyzer     → architecture.md
├── Task: deploy-analyzer   → deployment.md
└── Task: workflow-analyzer → workflows.md

Phase 3: INDEX Generation (Task, sonnet)
└── Task: index-generator
    ├── Reads all section docs
    └── Generates INDEX.md

Phase 4: Verification (main session)
└── Check file existence only (~10 lines)
    ls .ai-docs/
```

### Model Selection by Task Type

| Task Type | Model | Reason |
|-----------|-------|--------|
| File counting, structure scan | haiku | Simple extraction |
| Code pattern extraction | haiku | Mechanical search |
| Schema/endpoint listing | haiku | Structured extraction |
| Document writing | sonnet | Balance of quality/cost |
| INDEX generation | sonnet | Needs summarization |
| Complex architecture | opus | Only if truly complex |

### Task Distribution by Project Size

| Size | Files | Strategy |
|------|-------|----------|
| Small | < 50 | Single Task (sonnet) for all docs |
| Medium | 50-200 | 2-3 parallel Tasks + INDEX Task |
| Large | 200-500 | 5-6 parallel Tasks + INDEX Task |
| Very Large | 500+ | 10+ Tasks + hierarchical INDEX |

### Very Large Project: Hierarchical Tasks

When a section (e.g., API) has 50+ endpoints:

```
Phase 2-A: Detail Analysis (parallel, haiku)
├── api/auth-task    → api/auth.md
├── api/users-task   → api/users.md
└── api/products-task → api/products.md

Phase 2-B: Section INDEX (after 2-A, sonnet)
└── api-index-task → api/INDEX.md

Phase 3: Master INDEX (sonnet)
└── index-task → INDEX.md (reads section INDEXes only)
```

### Main Session Context Budget

| Phase | Lines Read | Notes |
|-------|------------|-------|
| Phase 1 | ~50 | package.json + dir structure |
| Phase 2 | 0 | Tasks handle everything |
| Phase 3 | 0 | Task handles INDEX |
| Phase 4 | ~10 | File list only |
| **Total** | **~60** | Minimal context usage |

### Task Prompts

**Section Doc Task:**
```
Analyze {section} and generate documentation.

Project: {project_path}
Scope: {file_patterns}
Output: {output_path}

Requirements:
- Write in English, use tables over prose
- Keep under {line_limit} lines
- Include all {items} found

Write output directly to {output_path}.
```

**INDEX Generation Task:**
```
Generate master INDEX.md for AI documentation.

Project: {project_path}
Docs folder: {docs_path}

Steps:
1. Read all .md files in {docs_path}
2. Extract: tech stack, structure, key files, commands
3. Create INDEX.md with:
   - Quick reference table
   - Project structure (annotated)
   - Key commands table
   - Documentation map with line counts
   - Critical files table

Keep INDEX.md under 150 lines.
Write to {docs_path}/INDEX.md.
```

## Generation Process

### 1. Analyze Project

Explore codebase to gather:
- Package manager and dependencies (package.json, requirements.txt, etc.)
- Framework and tech stack
- Directory structure
- Entry points and main files
- Database schema (if exists)
- API routes (if exists)
- Environment variables
- Build/deploy scripts

### 2. Generate INDEX.md

The INDEX is the AI's entry point. Must contain:

```markdown
# [Project Name] - AI Documentation

> One-line project description

## Quick Reference

| Aspect | Value |
|--------|-------|
| Framework | Next.js 16 / Django 5 / etc. |
| Language | TypeScript / Python / etc. |
| Database | PostgreSQL / MongoDB / etc. |
| Auth | Supabase Auth / JWT / etc. |

## Project Structure

[Directory tree with purpose annotations]

## Key Commands

| Task | Command |
|------|---------|
| Dev server | `npm run dev` |
| Build | `npm run build` |
| Test | `npm test` |
| Deploy | `npm run deploy` |

## Documentation Map

| Doc | When to Read |
|-----|--------------|
| [architecture.md](architecture.md) | Understanding tech stack, patterns |
| [api.md](api.md) | Working with API endpoints |
| [database.md](database.md) | Database operations, schema changes |
| [deployment.md](deployment.md) | Build, deploy, environment setup |
| [workflows.md](workflows.md) | Business logic, user flows |

## Critical Files

| File | Purpose |
|------|---------|
| `src/app/page.tsx` | Main entry point |
| `src/lib/db.ts` | Database client |
| ... | ... |
```

### 3. Generate Section Documents

Each document follows this pattern:

**architecture.md**
- Tech stack with versions
- Directory structure with purposes
- Design patterns used
- Component relationships
- State management approach

**api.md**
- All endpoints with methods
- Request/response schemas
- Authentication requirements
- Error codes
- Example requests

**database.md**
- All tables/collections
- Column types and constraints
- Relationships (FK, indexes)
- Common queries
- Migration commands

**deployment.md**
- Environment variables (with descriptions, not values)
- Build process
- Deploy commands
- CI/CD pipeline
- Hosting details

**workflows.md**
- User authentication flow
- Core business processes
- Data flow diagrams (ASCII)
- Error handling patterns

## Writing Guidelines

### Token Efficiency
- Use tables over prose
- Use code blocks for examples
- Avoid redundancy between documents
- Link instead of duplicate

### Completeness
- Every endpoint documented
- Every table documented
- Every env var documented
- Every command documented

### Actionability
- AI can run commands directly
- AI can write code matching patterns
- AI can debug with this info
- AI can deploy with this info

## Example: Minimal INDEX.md

```markdown
# MyApp - AI Documentation

> E-commerce platform with React frontend and Node.js API

## Quick Reference

| Aspect | Value |
|--------|-------|
| Frontend | React 18, TypeScript, Tailwind |
| Backend | Node.js 20, Express, Prisma |
| Database | PostgreSQL 15 |
| Auth | JWT + refresh tokens |

## Structure

```
src/
├── client/        # React SPA
│   ├── components/
│   ├── pages/
│   └── hooks/
├── server/        # Express API
│   ├── routes/
│   ├── services/
│   └── middleware/
└── shared/        # Shared types
```

## Commands

| Task | Command |
|------|---------|
| Dev | `npm run dev` |
| Build | `npm run build` |
| Test | `npm test` |
| Migrate | `npx prisma migrate dev` |

## Docs

- [architecture.md](architecture.md) - Tech decisions, patterns
- [api.md](api.md) - All 24 endpoints
- [database.md](database.md) - 12 tables, relationships
- [deployment.md](deployment.md) - Vercel + Railway setup
```

## Updating Docs

When project changes:
1. Identify affected documents
2. Update only changed sections
3. Keep INDEX.md current
4. Verify all links work
