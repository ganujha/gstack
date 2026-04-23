# INDEX.md — gstack Deep Understanding Materials

Complete reading guide for the generated learning package.

---

## Quick Start

**If you have 30 minutes**: Read these three files in order:
1. `gstack/00-system-overview.md` — What it is (5 min)
2. `gstack/01-architecture-map.md` — How it's built (15 min)
3. `gstack/09-teaching-note.md` §Engineer version — How to explain it (10 min)

**If you have 2 hours**: Follow the 2-hour path in `gstack/03-codebase-walkthrough.md`

**If you have 5 days**: Follow `_my-learning-sequence.md`

---

## All Materials

### Per-Repository Materials: gstack/

| File | Purpose | Read When |
|------|---------|-----------|
| `gstack/raw-findings.md` | Raw evidence collected during analysis | Reference: source files, state locations, commands |
| `gstack/evidence-table.md` | Claim → Evidence → Confidence → Verify | Fact-checking architectural claims |
| `gstack/00-system-overview.md` | BLUF, problem, users, subsystems | First read; orientation |
| `gstack/01-architecture-map.md` | Architecture, data flow, control flow, diagrams | Understanding the system structure |
| `gstack/02-agent-tool-memory-analysis.md` | Agent loop, tools, memory, sessions, context | Agentic systems analysis |
| `gstack/03-codebase-walkthrough.md` | Top 20 files, reading order, time-boxed paths | Starting code exploration |
| `gstack/04-design-patterns-and-antipatterns.md` | Strong patterns, risks, failure modes, suggestions | Code review and improvement |
| `gstack/05-interview-extraction.md` | 30 interview questions with answer bullets | Interview prep |
| `gstack/06-glossary.md` | Project vocabulary, internal terms | Reference while reading code |
| `gstack/07-open-questions.md` | Unresolved questions, assumptions, verification steps | Deeper investigation |
| `gstack/08-runbook-for-learning.md` | Exact commands, what to observe, safe experiments | Hands-on exploration |
| `gstack/09-teaching-note.md` | Three audience versions (engineer, staff+, CTO) | Explaining to others |
| `gstack/10-foundation-stack.md` | Technologies, frameworks, runtime, trade-offs | Technology evaluation |
| `gstack/11-component-decomposition.md` | Bottom-up layer analysis, core vs. replaceable | Architecture deep dive |

### Cross-Cutting Materials

| File | Purpose | Read When |
|------|---------|-----------|
| `_cross-repo-comparison.md` | gstack vs. common AI agent patterns; internal subsystem comparison | Comparative analysis |
| `_best-patterns-playbook.md` | 15 reusable patterns extracted from gstack | Applying patterns to new systems |
| `_my-learning-sequence.md` | 5-day structured study plan with experiments | Self-guided learning |
| `INDEX.md` | This file | Navigation |

---

## Reading Paths by Goal

### "I need to understand gstack for an interview"
1. `gstack/00-system-overview.md`
2. `gstack/01-architecture-map.md`
3. `gstack/05-interview-extraction.md` (full)
4. `gstack/04-design-patterns-and-antipatterns.md` (strong patterns + suggestions)
5. `gstack/09-teaching-note.md`

### "I need to contribute to gstack"
1. `gstack/03-codebase-walkthrough.md` (30-min path)
2. `gstack/08-runbook-for-learning.md` (experiments 1-5)
3. `gstack/10-foundation-stack.md`
4. `gstack/11-component-decomposition.md`
5. `gstack/07-open-questions.md`

### "I need to evaluate gstack for adoption"
1. `gstack/00-system-overview.md`
2. `gstack/09-teaching-note.md` (CTO version)
3. `gstack/04-design-patterns-and-antipatterns.md` (reliability risks + suggestions)
4. `_cross-repo-comparison.md` (production readiness table)
5. `gstack/10-foundation-stack.md`

### "I want to extract patterns for my own AI system"
1. `_best-patterns-playbook.md` (full)
2. `gstack/02-agent-tool-memory-analysis.md`
3. `gstack/04-design-patterns-and-antipatterns.md` (strong patterns only)
4. `_cross-repo-comparison.md`

### "I want to understand the security architecture"
1. `gstack/01-architecture-map.md` §Security diagrams
2. `gstack/02-agent-tool-memory-analysis.md` §Determinism vs. discretion
3. `gstack/04-design-patterns-and-antipatterns.md` §Security risks
4. `gstack/05-interview-extraction.md` Q3 (design 6-layer security)
5. `_best-patterns-playbook.md` patterns 11-12

### "I want to understand how the skill/prompt system works"
1. `gstack/00-system-overview.md` §What makes this interesting
2. `gstack/01-architecture-map.md` §Skill Template Compilation Pipeline (diagram)
3. `gstack/11-component-decomposition.md` §Skill Generation Service
4. `gstack/05-interview-extraction.md` Q2 (prompt distribution system)
5. `_best-patterns-playbook.md` patterns 2-3

---

## Key Concepts Quick Reference

| Concept | Defined in | Code location |
|---------|-----------|---------------|
| Browse daemon | `06-glossary.md` | `browse/src/server.ts` |
| SKILL.md template | `06-glossary.md` | Any `*/SKILL.md.tmpl` |
| @ref | `06-glossary.md` | `browse/src/snapshot.ts` |
| Preamble tier | `06-glossary.md` | `scripts/resolvers/preamble.ts` |
| HostConfig | `06-glossary.md` | `scripts/host-config.ts` |
| Resolver | `06-glossary.md` | `scripts/resolvers/index.ts` |
| Canary (security) | `06-glossary.md` | `browse/src/security.ts` |
| Touchfile | `06-glossary.md` | `test/helpers/touchfiles.ts` |
| Gate tier (testing) | `06-glossary.md` | `test/helpers/touchfiles.ts` E2E_TIERS |
| Boil the Lake | `06-glossary.md` | `ETHOS.md` |
| combineVerdict | `06-glossary.md` | `browse/src/security.ts` |
| Eval store | `06-glossary.md` | `test/helpers/eval-store.ts` |
| LLM judge | `06-glossary.md` | `test/helpers/llm-judge.ts` |
| Fix-first review | `06-glossary.md` | `review/SKILL.md.tmpl` |
| Dual-listener | `01-architecture-map.md` | `browse/src/server.ts` |
