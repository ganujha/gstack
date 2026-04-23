# _my-learning-sequence.md — gstack Study Plan

A structured 5-day learning plan to deeply understand the gstack codebase.

---

## Day 1: Read — Understand What and Why

**Goal**: Build a complete mental model of what the system does and why it was built this way.

### Morning (90 min): Philosophy and Overview

1. `ETHOS.md` — Read fully. Internalize "Boil the Lake" and "Search Before Building." These are the lens for every design decision. (20 min)
2. `ARCHITECTURE.md` — Read fully. Especially the "Why Bun" and "Daemon Model Rationale" sections. (20 min)
3. `CLAUDE.md` — Read the project structure section, testing section, and skill system sections. Skip the per-team-member instructions. (25 min)
4. `00-system-overview.md` (this docs folder) — Quick orientation. (10 min)
5. `01-architecture-map.md` (this docs folder) — Study the diagrams. (15 min)

### Afternoon (90 min): The Product (Skills)

6. `ship/SKILL.md` — Read the full generated skill (not the template). This is what Claude sees. Notice the preamble, the 19 steps, the stopping conditions. (30 min)
7. `review/SKILL.md` — Read the first 100 lines. Notice the 4-tier preamble structure and the fix-first pattern. (15 min)
8. `investigate/SKILL.md.tmpl` — Read fully. Notice the Iron Law, the 3-phase investigation, the freeze boundary. (20 min)
9. `autoplan/SKILL.md.tmpl` — Read the first 80 lines. Notice the sequential CEO→Design→Eng→DX pipeline and the decision taxonomy. (15 min)
10. `browse/SKILL.md` — Read fully (it's short). The $B command reference and QA patterns. (10 min)

### Evening (30 min): Component Decomposition Review

11. `11-component-decomposition.md` — Read the full decomposition. (20 min)
12. `10-foundation-stack.md` — Scan the technology table. (10 min)

**Day 1 checkpoint**: Can you explain to someone else what gstack is, what a skill is, and why the browse daemon uses a persistent architecture?

---

## Day 2: Run — Observe the System in Action

**Goal**: Validate mental model against actual system behavior.

### Setup (30 min)

```bash
cd /home/user/gstack
bun install
# Build browse binary
bun run build 2>&1 | tail -20
```

### Core Experiments (2 hours)

**Experiment 1: Skill generation pipeline** (30 min)
```bash
# See what a template looks like
cat browse/SKILL.md.tmpl | head -50

# Regenerate and observe
bun run gen:skill-docs -- --skill browse --dry-run

# Compare template vs. output
diff <(grep '{{' browse/SKILL.md.tmpl) <(echo "should be empty in output")
```
Note: What placeholders exist? How many lines does PREAMBLE expand to?

**Experiment 2: Browse daemon lifecycle** (30 min)
```bash
# Start daemon implicitly
./browse/dist/browse goto https://example.com

# Observe state
cat ~/.gstack/browse.json

# Inspect the actual running server
PORT=$(cat ~/.gstack/browse.json | python3 -c "import sys,json; print(json.load(sys.stdin)['port'])")
curl -s http://127.0.0.1:$PORT/health

# Snapshot the page
./browse/dist/browse snapshot -i

# Stop
./browse/dist/browse stop
cat ~/.gstack/browse.json 2>/dev/null || echo "State cleared"
```
Note: What does browse.json look like? What does snapshot output look like? How are @refs assigned?

**Experiment 3: Free test suite** (15 min)
```bash
bun test test/skill-validation.test.ts 2>&1 | head -50
bun test test/gen-skill-docs.test.ts 2>&1 | head -30
bun run skill:check 2>&1 | head -40
```
Note: What exactly do free tests validate? What would fail if you introduced a bug?

**Experiment 4: State and config** (15 min)
```bash
ls ~/.gstack/ 2>/dev/null || echo "Fresh install"
cat ~/.gstack/config.yaml 2>/dev/null || echo "No config"
ls ~/.gstack/security/ 2>/dev/null || echo "No security state"
bun run eval:list 2>&1 | head -20
```
Note: What state exists on your machine? What's project-scoped vs. global?

**Experiment 5: Resolver system** (30 min)
```bash
# See all resolver files and sizes
ls -lh scripts/resolvers/*.ts | sort -k5 -rh

# Read the resolver registry
cat scripts/resolvers/index.ts | head -60

# Read preamble tier system
cat scripts/resolvers/preamble.ts

# Read one preamble generator
cat scripts/resolvers/preamble/generate-lake-intro.ts
```
Note: How does the registry map placeholder names to functions? What does a generator return?

**Evening (30 min): Notes and questions**

Write down:
- 3 things that surprised you
- 3 things that confirmed your mental model
- 3 open questions you now have

Cross-reference with `07-open-questions.md`.

---

## Day 3: Compare — Understand Architecture Choices

**Goal**: Deeply understand why specific architectural choices were made by comparing alternatives.

### Morning (90 min): Security Architecture

1. Read `browse/src/security.ts` — focus on `combineVerdict()` and `THRESHOLDS`
2. Read `browse/src/content-security.ts` — focus on ARIA injection patterns and hidden element detection
3. Read `browse/src/tunnel-denial-log.ts` — note rate limiting and async writes
4. Read CLAUDE.md §Sidebar security stack — the 6-layer table
5. Ask: "What would happen if only L4 (ML classifier) existed?" — trace the FP/FN failure modes
6. Ask: "Why is port separation stronger than header checks?" — try to break a header-based check mentally

### Afternoon (90 min): Test Infrastructure

1. Read `test/helpers/session-runner.ts` — fully understand how `claude -p` is spawned and parsed
2. Read `test/helpers/touchfiles.ts` — scan the `E2E_TOUCHFILES` array; pick 3 test entries and trace their dependencies
3. Read `test/helpers/llm-judge.ts` — understand the rubric and the 429 retry
4. Run: `bun run eval:select` — see which tests your current branch would trigger
5. Ask: "What test is not covered that should be?" — compare against `04-design-patterns-and-antipatterns.md` gaps

### Evening (30 min): Design Docs Deep Dive

Pick one design doc from `docs/designs/` and read it fully:
- `GSTACK_BROWSER_V0.md` — browser product thesis
- `ML_PROMPT_INJECTION_KILLER.md` — security architecture genesis
- `GCOMPACTION.md` — context efficiency problem

Ask: What problem was being solved? What alternatives were considered? What was built?

---

## Day 4: Explain — Teach It Out Loud

**Goal**: Solidify understanding by explaining the system in your own words.

### Exercise 1: The Architecture Talk (30 min)

Without looking at notes, explain to an imaginary colleague:
- What gstack is (1 sentence)
- How the browse daemon works (30 seconds)
- How skills are generated (30 seconds)
- What the preamble tier system does (30 seconds)
- What the 6-layer security does and why (1 minute)

Check your explanation against `09-teaching-note.md`.

### Exercise 2: Answer 5 Interview Questions (60 min)

Pick 5 questions from `05-interview-extraction.md` and answer them without looking at the answers. Then compare. For each discrepancy, trace back to the source file.

### Exercise 3: Trace a Full Request (30 min)

Mentally trace: "User runs `$B click @e3` after the browser is already running."

Trace through:
1. cli.ts receives args
2. readState() — what does browse.json contain?
3. isServerHealthy() — HTTP GET /health
4. POST /command to server
5. server.ts routes to click handler
6. content-security.ts checks (L1-L3)
7. Response returned
8. cli.ts prints result

### Exercise 4: The One-Page Summary (20 min)

Write a one-page (300 word) summary of gstack's architecture that you could send to a fellow engineer who's never heard of it. No looking at existing summaries.

---

## Day 5: Challenge — Find Gaps and Propose Improvements

**Goal**: Move from understanding to critical evaluation.

### Morning (90 min): Identify Gaps

1. Read `07-open-questions.md` — which questions can you now answer? Which remain?
2. For each open question, spend 5 minutes trying to find the answer in source code
3. Read `04-design-patterns-and-antipatterns.md` §suggestions for hardening
4. Prioritize: which 3 hardening suggestions would you implement first and why?

### Afternoon (90 min): Design a Concrete Improvement

Pick one gap from the antipatterns document and design a concrete fix:

**Option A: Atomic state file writes for browse.json**
- Current: may have race condition on concurrent CLI invocations
- Fix: write to `.tmp` → `rename()` (atomic on POSIX)
- Trace through: what changes in cli.ts / server.ts?
- Estimate effort: <30 lines of code?

**Option B: Per-test cost caps in touchfiles.ts**
- Current: no spend limit enforcement per test
- Fix: add `maxCostCents: number` to each E2E_TOUCHFILES entry; check before execution
- Trace through: what changes in eval infrastructure?
- Estimate effort: 50-100 lines?

**Option C: Security red-team E2E test**
- Current: no automated test that verifies injection is blocked
- Fix: add `security-e2e.test.ts` with known injection payloads; assert verdict is `block`
- Trace through: what fixture, what session-runner call, what assertion?
- Estimate effort: 100-150 lines?

### Evening (30 min): Final Synthesis

Complete the `07-open-questions.md` file with your findings:
- Mark each question as RESOLVED / PARTIALLY RESOLVED / STILL OPEN
- Add evidence for resolved questions
- Add new questions you discovered

Write a "What I'd improve" list with 5 concrete suggestions, ranked by impact.

---

## Daily Reading Checklist

| Day | Files Read | Experiments Run | Key Insight |
|-----|-----------|-----------------|-------------|
| 1 | ETHOS.md, ARCHITECTURE.md, CLAUDE.md, ship/SKILL.md, review/SKILL.md, investigate/SKILL.md.tmpl | None | |
| 2 | browse/src/cli.ts, scripts/resolvers/preamble.ts, scripts/resolvers/index.ts | Browse daemon, skill gen, free tests | |
| 3 | browse/src/security.ts, content-security.ts, test/helpers/session-runner.ts, touchfiles.ts | eval:select, one design doc | |
| 4 | 05-interview-extraction.md | Architecture talk, 5 interview Qs, request trace | |
| 5 | 07-open-questions.md, 04-design-patterns-and-antipatterns.md | Design improvement | |
