# ai-docs

Claude Code plugin for generating AI-optimized project documentation.

Current active workflow: **v1.5**.

v1.5 keeps the proven v1 12-document output, but fixes v1's main weakness: fixed writer/reviewer partitioning. The scout phase measures repository scale and complexity, writes a workload matrix, and assigns Claude agents dynamically.

v2/v3/v4 experiments are not active in this repository.

## Output

```text
ai-docs/
├── SPEC.md
├── .skeleton.md
├── 00_INDEX.md
├── 01_ENVIRONMENT.md
├── 02_DEPENDENCIES.md
├── 03_ARCHITECTURE.md
├── 04_STRUCTURE.md
├── 05_DATA_MODELS.md
├── 06_API.md
├── 07_BUSINESS_LOGIC.md
├── 08_DEBUG.md
├── 09_STANDARDS.md
├── 10_WARNINGS.md
└── 11_TODO.md
```

## Flow

```text
Scout team
  -> .skeleton.md
     -> shared facts
     -> cross-reference map
     -> workload matrix
     -> agent assignment plan
  -> workload-based writer team
  -> workload-based reviewer team
  -> cross-checker
  -> 00_INDEX.md
```

The 12 files are the output contract, not the agent partition. Heavy documents such as `05_DATA_MODELS.md`, `06_API.md`, and `07_BUSINESS_LOGIC.md` must be split or assigned dedicated owners when the workload matrix says they are heavy/critical.

## Installation

```bash
mkdir -p ~/.claude/plugins/marketplaces
cd ~/.claude/plugins/marketplaces
git clone https://github.com/aminpiano/ai-docs.git
```

After installation, restart Claude Code. The `ai-project-docs` skill is available.

## Structure

```text
ai-docs/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── ai-project-docs/
│       ├── SKILL.md
│       └── references/
│           ├── full-mode.md
│           ├── update-mode.md
│           └── default-spec.md
├── SPEC.md
└── README.md
```

| File | Purpose |
|------|---------|
| `skills/ai-project-docs/SKILL.md` | Skill entry point and orchestration rules |
| `skills/ai-project-docs/references/full-mode.md` | Full generation procedure |
| `skills/ai-project-docs/references/update-mode.md` | Existing-doc update procedure |
| `skills/ai-project-docs/references/default-spec.md` | Fallback SPEC generator |
| `SPEC.md` | Reusable generation specification |
