## §DEFAULT-SPEC

If `ai-docs/SPEC.md` does not exist, create it before proceeding.

**Recommended approach**: If this repository (ai-docs) is available, copy the canonical `SPEC.md` from it. This ensures consistency and completeness.

**Fallback** (if canonical SPEC is not available): Generate a SPEC covering these sections:

1. **§0 Hard Rules**: Evidence-based writing, Evidence sections, output format (UTF-8, Mermaid fences, code blocks), metadata (date + commit hash), cross-reference principle (canonical source priority table), verification checklist (the 8 items below), INDEX last, security (no secrets), section numbering (§N format). The 8-item verification checklist that reviewers and cross-checker must apply to every document:
   1. File exists with correct filename
   2. All §N section headers match skeleton's numbering scheme
   3. `## Evidence` section exists listing source files
   4. Cross-references (`→ Detail:`) resolve to existing sections
   5. Shared Facts (Value Facts + Classification Map) match skeleton
   6. Metadata line present (`> Generated: ...`)
   7. Anti-patterns in 09 reference TODO IDs from 11
   8. TODO registry in 11 includes all skeleton items
2. **§1 Execution Model**: 3-Phase (scout → write team → review team), Phase 0 skeleton structure (Value Facts, Classification Map, Cross-Reference Map, TODO Registry with ID rules, Section Numbering), Phase 1 parallel writing rules (writers must use skeleton classifications verbatim — never re-derive), Phase 2 parallel review + cross-check (cross-checker must self-verify their own 00_INDEX output)
3. **§2 File List**: 12 mandatory files (00_INDEX through 11_TODO) with phase assignment
4. **§3 INDEX Rules**: Project Identity, Quick Start (explicit all steps), Document Map with Common Task Routing (8 default rows), Critical Warnings Summary (3-5 lines with cross-refs). INDEX uses §0-N namespace (§0-1 through §0-4) to avoid collision with 03_ARCHITECTURE's §3-N
5. **§4 Common Principles**: AI audience, format priority (table > bullet > code > prose), N/A handling, Evidence, empty item consolidation, directory tree exclusions, section numbering compliance, ownership principle
6. **§5 Per-Document Content**: Detailed requirements for all 11 content documents including ownership restrictions (04, 06), boundary rules (06↔07), TODO Registry compliance (09, 11), priority system (P0-P3 with tiebreaker)
7. **§6 Update Policy**: Regeneration triggers, method selection (full vs update mode), incremental update constraints, temporary file cleanup
