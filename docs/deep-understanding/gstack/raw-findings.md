# raw-findings.md — gstack Repository

## Repo Map

```
/home/user/gstack/
├── .github/workflows/      — 6 CI workflow files
├── bin/                    — 38 shell/TS CLI utilities
├── browse/                 — Headless browser daemon (Bun + TypeScript)
│   ├── src/                — 33 TypeScript source files
│   ├── test/               — Integration tests + fixtures
│   └── dist/               — Compiled binary (gitignored but tracked historically)
├── design/                 — Design binary CLI (GPT Image API)
├── docs/designs/           — 15+ design documents
├── extension/              — Chrome extension (sidepanel + background + content)
├── hosts/                  — 10 host config TypeScript files
├── lib/                    — Shared libraries (worktree.ts)
├── make-pdf/               — PDF generation binary
├── openclaw/skills/        — 4 native OpenClaw skills
├── scripts/                — 20+ build + DX scripts
│   └── resolvers/          — 19 resolver modules (~200K total lines)
├── supabase/               — Supabase backend (functions + migrations)
├── test/                   — 20+ test files + helpers
│   └── helpers/            — session-runner, eval-store, llm-judge, touchfiles, skill-parser
├── <40 skill directories>  — Each with SKILL.md + SKILL.md.tmpl
├── ARCHITECTURE.md         — Daemon design + dual-listener docs
├── CHANGELOG.md            — 321K release history
├── CLAUDE.md               — Developer instructions (this repo)
├── ETHOS.md                — Founder philosophy (Garry Tan / YC)
├── SKILL.md / SKILL.md.tmpl— Root gstack discovery skill
├── TODOS.md                — 74K backlog
├── VERSION                 — Current version (1.6.3.0)
└── package.json            — Bun workspace config
```

## Entrypoints

| Entrypoint | Type | Purpose |
|------------|------|---------|
| `browse/src/cli.ts` | Binary entrypoint | CLI for all browse commands |
| `browse/src/server.ts` | HTTP server | Persistent Chromium daemon (Bun.serve) |
| `browse/src/snapshot.ts` | Module | ARIA snapshot capture |
| `scripts/gen-skill-docs.ts` | Build script | Template → SKILL.md generator |
| `setup` | Shell script (1012 lines) | Full installation orchestrator |
| `bin/gstack-*` | Shell/TS utilities | Per-feature CLI commands |
| `test/*.test.ts` | Bun test files | Unit/E2E test entry |
| `hosts/index.ts` | Module | Host registry (10 AI agents) |

## Key Files

| File | Size | Importance |
|------|------|------------|
| `browse/src/server.ts` | 124K | Core: HTTP server, security, Chromium lifecycle |
| `browse/src/cli.ts` | ~40K | Core: Command dispatch, state management |
| `scripts/resolvers/review.ts` | 54K | Skill content: CEO/Eng/Design review specs |
| `scripts/resolvers/design.ts` | 58K | Skill content: Design review checklist |
| `scripts/resolvers/testing.ts` | 27K | Skill content: Test framework detection |
| `scripts/resolvers/utility.ts` | 17K | Skill content: Shared placeholders |
| `scripts/resolvers/review-army.ts` | 11K | Skill content: Multi-reviewer dispatch |
| `scripts/gen-skill-docs.ts` | 28K | Build: Template processing pipeline |
| `test/helpers/touchfiles.ts` | 26K | Testing: Diff-based test selection |
| `test/helpers/eval-store.ts` | 25K | Testing: Eval persistence + comparison |
| `ship/SKILL.md.tmpl` | 42K | Skill: Full shipping workflow |
| `CHANGELOG.md` | 321K | History: Full release notes |

## Configuration Files

| File | Purpose |
|------|---------|
| `package.json` | Bun workspace, version 1.6.3.0, build scripts |
| `slop-scan.config.json` | AI code quality scanner exclusions |
| `actionlint.yaml` | GitHub Actions linting rules |
| `conductor.json` | Conductor workspace config |
| `.env.example` | Only `ANTHROPIC_API_KEY` needed |
| `hosts/*.ts` | Per-AI-agent skill distribution configs |
| `scripts/jargon-list.json` | Auto-glossed terms in skill output |
| `~/.gstack/config.yaml` | Runtime config (skill_prefix, team_mode, etc.) |
| `~/.gstack/browse.json` | Running server state (pid, port, token) |
| `~/.gstack/security/session-state.json` | Security layer cross-process state |

## Commands Discovered

```bash
# Build
bun install
bun run build              # gen docs + compile binaries
bun run gen:skill-docs     # regenerate SKILL.md from .tmpl files

# Test
bun test                   # free, <2s (skill validation + browse integration)
bun run test:evals         # paid, diff-based ($4 max)
bun run test:evals:all     # all evals regardless of diff
bun run test:gate          # gate tier only (CI default)
bun run test:e2e           # E2E only ($3.85 max)
bun run test:e2e:all       # all E2E

# Dev
bun run dev <cmd>          # run CLI in dev mode
bun run dev:skill          # watch mode: auto-regen + validate
bun run skill:check        # health dashboard for all skills

# Eval
bun run eval:select        # preview which tests would run
bun run eval:list          # list all eval runs
bun run eval:compare       # compare two eval runs
bun run eval:summary       # aggregate stats

# Code quality
bun run slop               # full slop-scan report
bun run slop:diff          # slop findings in changed files only

# Browse daemon
$B goto https://example.com
$B snapshot -i
$B click @e1
$B screenshot --out /tmp/shot.png

# Install
./setup                    # full install (builds + links skills)
./setup --team             # team mode install
./setup --prefix           # use gstack-* prefix for skill names
./setup --no-prefix        # flat skill names
```

## Important Classes/Functions/Modules

### browse/src/server.ts
- `Bun.serve()` — dual-listener HTTP setup (local + tunnel port)
- `handleCommand()` — routes HTTP POST to command handlers
- `SecurityLayer` — coordinates ML classifiers + canary injection
- `SidebarModel pickSidebarModel()` — Opus vs. Sonnet routing for sidebar
- State: `~/.gstack/browse.json` (pid, port, token, serverPath, binaryVersion)

### browse/src/cli.ts
- `readState()` — parses `.gstack/browse.json`
- `isServerHealthy()` — HTTP GET /health liveness check
- `startServer()` — spawns daemon subprocess (detached)
- `acquireServerLock()` — atomic file lock (prevents TOCTOU race on startup)
- `killServer()` — SIGTERM → wait 2s → SIGKILL
- `runCommand()` — main dispatch (parses args, calls server API)

### browse/src/snapshot.ts
- `takeSnapshot()` — Playwright ARIA tree → @ref-annotated text
- `SNAPSHOT_FLAGS` — 9 documented flag definitions

### scripts/gen-skill-docs.ts
- `generateSkillDocs(hosts, options)` — main pipeline
- `resolveTemplate(tmpl, context)` — placeholder expansion
- `transformFrontmatter(host, fm)` — host-aware YAML transforms

### scripts/resolvers/preamble.ts
- `generatePreamble(tier, context)` — tier 1-4 preamble composition
- Imports 22 preamble generator functions from `preamble/` subdirectory

### test/helpers/session-runner.ts
- `runSession(prompt, options)` — spawns `claude -p`, streams NDJSON
- `parseNDJSON(lines)` — pure function: NDJSON → structured transcript
- Returns: `toolCalls, browseErrors, exitReason, costEstimate, model, firstResponseMs`

### test/helpers/llm-judge.ts
- `callJudge<T>(prompt)` — JSON extraction with 429 retry
- `judge(section, content)` — scores docs on clarity/completeness/actionability
- `outcomeJudge()` — planted-bug detection scorer

### lib/worktree.ts
- `WorktreeManager.createWorktree()` — isolated git worktree per test
- `WorktreeManager.harvest()` — diff collection after test
- Dedup index: persistent hash tracking via `getDedupPath()`

## Prompts / Rules / Skills Locations

| Location | What lives there |
|----------|-----------------|
| `<skill>/SKILL.md.tmpl` | Skill template source (hand-authored) |
| `<skill>/SKILL.md` | Generated skill prompt (multi-host capable) |
| `scripts/resolvers/*.ts` | Shared prompt sections (preamble, review, design, etc.) |
| `scripts/resolvers/preamble/` | 22 preamble section generators |
| `ETHOS.md` | Founder philosophy (used as grounding context) |
| `scripts/jargon-list.json` | Auto-glossed terms baked into preamble |
| `hosts/*.ts` | Per-host frontmatter transforms + path rewrites |

## Memory / State Locations

| State | Location | Persistence |
|-------|----------|-------------|
| Server state | `~/.gstack/browse.json` | Session (cleared on stop) |
| Security state | `~/.gstack/security/session-state.json` | Session |
| Security attack log | `~/.gstack/security/attempts.jsonl` | Persistent (rotates at 10MB) |
| Device salt | `~/.gstack/security/device-salt` | Persistent |
| Model cache | `~/.gstack/models/testsavant-small/` | Persistent |
| Eval results | `~/.gstack/projects/$SLUG/evals/` | Persistent |
| Learnings | `~/.gstack/learnings/` (inferred) | Persistent |
| User config | `~/.gstack/config.yaml` | Persistent |
| Git worktree temp | `/tmp/gstack-worktrees/` (inferred) | Ephemeral |
| Version marker | `~/.gstack/.last-setup-version` | Persistent |

## Tool Integration Locations

| Tool / Integration | Location |
|-------------------|---------|
| Playwright (Chromium) | `browse/src/server.ts`, `browse/src/browser-manager.ts` |
| Anthropic SDK | `test/helpers/llm-judge.ts`, `browse/src/server.ts` |
| ngrok tunnel | `browse/src/cli.ts` (`connect`, `pair-agent` commands) |
| GitHub Actions CI | `.github/workflows/` |
| Supabase | `supabase/functions/`, `supabase/migrations/` |
| Chrome extension | `extension/` |
| ML classifiers | `browse/src/security-classifier.ts` (ONNX, @huggingface/transformers) |
| Git worktrees | `lib/worktree.ts` |
| Codex CLI | `codex/SKILL.md.tmpl`, `test/codex-e2e.test.ts` |
| GPT Image API | `design/src/` |

## Runtime Assumptions

- Bun >= 1.0.0 (required — uses `Bun.serve`, `Bun.build --compile`, native SQLite)
- Playwright Chromium installed separately via `bunx playwright install`
- macOS ARM64: requires codesigning after binary compilation
- Windows: falls back to Node.js for server subprocess (Bun pipe issue with Chromium)
- `ANTHROPIC_API_KEY` env var required for LLM-judge evals
- Codex auth: `~/.codex/auth.json` (or CODEX_API_KEY / OPENAI_API_KEY)
- ngrok optional (for pair-agent tunnel mode)

## Uncertainties

- Exact Supabase schema and what backend functions power (analytics? telemetry?)
- Whether `gstack-telemetry-sync` sends data to external server or is local-only
- Full extent of `gbrain` integration (external service vs. local inference)
- Whether `conductor.json` is used by an external Conductor product or internal
- Complete implementation of `gstack-learnings-*` persistence format
- Whether `sidebar-agent.ts` (mentioned in CLAUDE.md) is inside browse/src or separate
