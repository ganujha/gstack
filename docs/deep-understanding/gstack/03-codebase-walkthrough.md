# 03-codebase-walkthrough.md — gstack

## Top 20 Most Important Files

| Priority | File | Why It Matters |
|----------|------|----------------|
| 1 | `browse/src/server.ts` | The core daemon: HTTP routing, Chromium lifecycle, security integration, tunnel topology |
| 2 | `browse/src/cli.ts` | The primary user interface: command dispatch, server lifecycle, pair-agent, headed mode |
| 3 | `scripts/gen-skill-docs.ts` | The build system brain: template → SKILL.md pipeline, host-aware transforms |
| 4 | `scripts/resolvers/preamble.ts` | Preamble composition architecture: how skill tiers are assembled |
| 5 | `hosts/index.ts` + `hosts/claude.ts` | Host abstraction: how gstack supports 10 different AI agents |
| 6 | `scripts/host-config.ts` | HostConfig interface: the contract all hosts implement |
| 7 | `browse/src/commands.ts` | The entire command surface: 50 commands with security annotations |
| 8 | `browse/src/snapshot.ts` | ARIA snapshot system: @ref assignment, flag definitions |
| 9 | `browse/src/security.ts` | 6-layer defense: canary injection, ML ensemble, threshold rules |
| 10 | `browse/src/content-security.ts` | DOM-level defense: datamarking, hidden element stripping |
| 11 | `ship/SKILL.md.tmpl` | The flagship skill: 42K template, 19-step workflow |
| 12 | `scripts/resolvers/review.ts` | CEO/Design/Eng/DX review specifications (54K of opinionated content) |
| 13 | `test/helpers/session-runner.ts` | How E2E tests work: claude -p subprocess model |
| 14 | `test/helpers/touchfiles.ts` | Diff-based test selection: which evals run per PR |
| 15 | `test/helpers/eval-store.ts` | Eval persistence and trend analysis |
| 16 | `test/helpers/llm-judge.ts` | LLM-as-judge quality scoring |
| 17 | `lib/worktree.ts` | Git worktree management for isolated test execution |
| 18 | `scripts/resolvers/preamble/generate-context-recovery.ts` | Context health and recovery instructions |
| 19 | `setup` | Full installation script: build, link, migrate |
| 20 | `ETHOS.md` | The philosophical grounding for all design decisions |

## Recommended Reading Order

### 30-Minute Understanding

**Goal**: Understand what gstack is, what it does, and why it's architecturally interesting.

1. **`ETHOS.md`** (5 min) — Read this first. "Boil the Lake" and "Search Before Building" are the philosophical lenses for every design decision. Understand why completeness is cheap now.

2. **`00-system-overview.md`** (this docs folder) (3 min) — One-page BLUF.

3. **`browse/src/commands.ts`** (5 min) — Scan the three command arrays (READ, WRITE, META). Count 50 commands. Notice `wrapUntrustedContent()` annotation — security is first-class.

4. **`ship/SKILL.md`** or **`review/SKILL.md`** (10 min) — Read a generated skill in full. This is what Claude sees. Notice the preamble section at the top, then the step-by-step workflow. This is the product.

5. **`ARCHITECTURE.md`** (7 min, first half) — The dual-listener topology and why a persistent daemon matters. The "Why Bun" section is concise and correct.

### 2-Hour Understanding

**Goal**: Understand how the system is built and how to modify it.

1. **`browse/src/cli.ts`** — Server lifecycle (`readState`, `isServerHealthy`, `startServer`, `acquireServerLock`). Follow the startup path mentally.

2. **`browse/src/server.ts`** (first 200 lines) — HTTP server setup, dual-listener config, route dispatch. Note the command allowlist for tunnel surface.

3. **`scripts/gen-skill-docs.ts`** — Read the main pipeline function. Understand how `{{PLACEHOLDERS}}` are resolved. Trace how frontmatter is transformed per host.

4. **`scripts/resolvers/preamble.ts`** — The tier system. Count the generators in each tier. Read `preamble/generate-lake-intro.ts` and `preamble/generate-context-recovery.ts` as examples.

5. **`hosts/claude.ts`** + **`scripts/host-config.ts`** — The HostConfig interface. Understand `frontmatter.mode: 'denylist'`, `linkingStrategy: 'real-dir-symlink'`.

6. **`browse/src/security.ts`** — The `combineVerdict()` function. The ensemble rule (2-of-3 ML at WARN → BLOCK). The canary deterministic path.

7. **`test/helpers/session-runner.ts`** — How `claude -p` is spawned, how NDJSON is parsed, what `SkillTestResult` contains.

8. **`test/helpers/touchfiles.ts`** — Scan `E2E_TOUCHFILES` array. Understand the `matchGlob()` logic. The `gate` vs `periodic` tier classification.

### 1-Day Understanding

**Goal**: Understand the entire system deeply enough to contribute.

**Morning: Skill system**
1. `scripts/gen-skill-docs.ts` — full file
2. `scripts/resolvers/index.ts` — resolver registry, all placeholder names
3. `scripts/resolvers/review.ts` — CEO + Eng + Design + DX review content (54K — skim sections, don't read every word)
4. `scripts/resolvers/utility.ts` — shared utilities
5. `scripts/resolvers/preamble/*.ts` — read all 22 generators
6. `hosts/*.ts` — all 10 host configs
7. A complete skill template: `autoplan/SKILL.md.tmpl` (shows coordinator pattern)
8. Another: `investigate/SKILL.md.tmpl` (shows freeze/scope discipline)

**Afternoon: Browse daemon**
1. `browse/src/server.ts` — full file (large, focus on route handlers and security integration)
2. `browse/src/cli.ts` — full file (server lifecycle, Windows edge cases)
3. `browse/src/snapshot.ts` — ARIA tree mechanics
4. `browse/src/security.ts` + `browse/src/content-security.ts` + `browse/src/tunnel-denial-log.ts`
5. `browse/src/browser-manager.ts` — Playwright tab management
6. `extension/sidepanel.js` (first 100 lines) — sidebar chat architecture

**Evening: Test + ops**
1. `test/helpers/` — all 5 core helpers
2. `test/skill-validation.test.ts` — what gets caught by free tests
3. `test/skill-e2e-ship.test.ts` — how a gate-tier E2E works
4. `lib/worktree.ts` — git isolation model
5. `setup` — full installation script
6. `.github/workflows/evals.yml` — CI pipeline structure
7. `docs/designs/GCOMPACTION.md` + `PLAN_TUNING_V1.md` — understand the future direction

## "Read This First / Second / Third" Guidance

### For understanding the AI skill system:
1. First: `ETHOS.md`
2. Second: Any generated `SKILL.md` (e.g., `ship/SKILL.md`)
3. Third: `scripts/gen-skill-docs.ts` + `scripts/resolvers/preamble.ts`

### For understanding the browser automation:
1. First: `ARCHITECTURE.md` §Daemon Model Rationale
2. Second: `browse/src/cli.ts` startup path
3. Third: `browse/src/server.ts` route handlers

### For understanding security:
1. First: `CLAUDE.md` §Sidebar security stack (the table)
2. Second: `browse/src/security.ts` `combineVerdict()`
3. Third: `browse/src/content-security.ts` L1-L3

### For understanding testing:
1. First: `CLAUDE.md` §Testing section
2. Second: `test/helpers/session-runner.ts`
3. Third: `test/helpers/touchfiles.ts` `E2E_TOUCHFILES`

### For contributing a new skill:
1. First: Any existing `*/SKILL.md.tmpl` as reference
2. Second: `scripts/resolvers/preamble.ts` (choose your tier)
3. Third: `test/helpers/touchfiles.ts` (add your skill's touchfile entry)
4. Fourth: `scripts/gen-skill-docs.ts` `--dry-run` to validate

## File Reading Sequence for Understanding Data Flow

```
1. package.json              — versions, scripts, entry points
2. hosts/index.ts            — supported AI agents
3. hosts/claude.ts           — primary host config
4. scripts/host-config.ts   — HostConfig interface spec
5. browse/src/commands.ts   — tool surface (50 commands)
6. browse/src/snapshot.ts   — @ref system
7. browse/src/cli.ts         — server lifecycle
8. browse/src/server.ts      — request handling
9. browse/src/security.ts    — defense layers
10. scripts/resolvers/preamble.ts — skill preamble
11. ship/SKILL.md.tmpl       — complete workflow template
12. test/helpers/session-runner.ts — how tests invoke Claude
13. test/helpers/touchfiles.ts    — diff-based test gating
14. test/helpers/eval-store.ts    — eval persistence
15. lib/worktree.ts              — git isolation
16. setup (shell script)          — installation logic
17. .github/workflows/evals.yml   — CI pipeline
```
