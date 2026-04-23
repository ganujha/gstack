# 10-foundation-stack.md — gstack

## Primary Languages Used

| Language | Evidence | Role |
|----------|----------|------|
| **TypeScript** | `browse/src/*.ts`, `scripts/*.ts`, `hosts/*.ts`, `test/**/*.ts` | Primary language for all non-shell code |
| **Bash/Shell** | `setup` (1012-line installer), `bin/gstack-*` utilities, `gstack-upgrade/migrations/*.sh` | Installation, CLI utilities, version migrations |
| **Markdown** | `*/SKILL.md`, `*/SKILL.md.tmpl`, `docs/**/*.md` | Skill prompts (the "product"), documentation |
| **JSON** | `package.json`, `conductor.json`, `slop-scan.config.json`, `scripts/jargon-list.json` | Configuration, metadata |
| **YAML** | `actionlint.yaml`, `.github/workflows/*.yml`, skill frontmatter | CI config, skill metadata |
| **JavaScript** | `extension/*.js` | Chrome extension (sidepanel, background, content scripts) |

---

## Package Managers and Build Tools

### Primary: Bun

- **Evidence**: `package.json` → `"engines": {"bun": ">=1.0.0"}`, `bun.lock`, all scripts use `bun run` / `bun test` / `bun build`
- **Responsibility**: Dependency management, TypeScript execution (no transpile step), binary compilation, test runner
- **Central or replaceable**: Central — the entire build pipeline depends on Bun's `build --compile` for producing self-contained binaries
- **Trade-offs**: Fast startup, native TypeScript, native SQLite; poor Windows Chromium pipe handling (requires Node.js fallback); smaller ecosystem than npm

### Compilation: `bun build --compile`

- **Evidence**: `setup` script: `bun build --compile browse/src/cli.ts --outfile browse/dist/browse`
- **Responsibility**: Produces single-binary executables (~58MB) with all code + runtime bundled
- **Trade-offs**: Platform-specific (Mach-O arm64 for macOS), requires codesign on macOS; no cross-compilation

### Test Runner: `bun test`

- **Evidence**: `package.json` scripts: `"test": "bun test"`, test files use `import {test, expect} from 'bun:test'`
- **Responsibility**: Free tests (<2s), skill validation, integration tests
- **Trade-offs**: No Jest compatibility layer needed; limited plugins vs. Vitest/Jest ecosystem

---

## Runtime Entrypoints

| Entrypoint | Command | Description |
|------------|---------|-------------|
| `browse/src/cli.ts` | `browse <cmd>` or `$B <cmd>` | Primary CLI for all browser commands |
| `browse/src/server.ts` | Spawned by cli.ts | HTTP server (Bun.serve), Chromium daemon |
| `scripts/gen-skill-docs.ts` | `bun run gen:skill-docs` | Build-time skill doc generation |
| `scripts/skill-check.ts` | `bun run skill:check` | Health dashboard |
| `setup` (bash) | `./setup` | One-time installation |
| `bin/gstack-*` | Various | Per-feature utility commands |
| `test/*.test.ts` | `bun test` | Test runner entry |
| `design/src/cli.ts` | `design <cmd>` | Design binary CLI |
| `make-pdf/src/` | `make-pdf` | PDF generation binary |

---

## Process Model

```
User machine:
  ├── claude (Claude Code, external binary) — interactive session
  │   └── reads SKILL.md from ~/.claude/skills/
  │   └── runs Bash: $B commands → HTTP
  │
  ├── browse (Bun compiled binary, detached daemon)
  │   ├── cli.ts — command dispatch, state management
  │   ├── server.ts — Bun.serve HTTP (two ports: local + tunnel)
  │   ├── Chromium subprocess (via Playwright)
  │   └── sidebar-agent subprocess (in headed mode only)
  │
  └── ngrok (optional, for pair-agent tunnel)
      └── forwards to browse daemon tunnel port only
```

**Key process properties**:
- Browse daemon is detached (continues after CLI exits)
- Singleton enforced by `~/.gstack/browse.json` + `wx` lock file
- 30-minute idle auto-shutdown
- Windows uses Node.js for the server subprocess (not Bun) due to Chromium pipe issues

---

## Agentic Framework(s) Used

| Framework | Evidence | Role |
|-----------|----------|------|
| **Claude Code (external)** | Users invoke `claude`, SKILL.md files use Claude-specific tools | Primary agent runtime; not built here, consumed by here |
| **Anthropic SDK** (`@anthropic-ai/sdk` 0.78.0) | `package.json`, `test/helpers/llm-judge.ts` | LLM-as-judge calls, sidebar model inference |
| **No custom agent loop** | Absence of loop/orchestration code | gstack extends Claude Code; doesn't implement its own |

gstack's "agent framework" contribution is the **skill system** (SKILL.md as system prompt) and the **browse daemon** (tool). The agent loop is Claude Code's built-in capability.

---

## API / Web Framework(s) Used

| Framework | Evidence | Role |
|-----------|----------|------|
| **Bun.serve** | `browse/src/server.ts` → `Bun.serve({port, fetch})` | HTTP server for browse daemon |
| No Express/Fastify/Hono | Absence from package.json | Bun's built-in HTTP handles all routing |

The HTTP routing in `server.ts` is hand-rolled: URL pattern matching in the `fetch` handler. This is appropriate for a fixed, small route set. No framework overhead.

---

## UI Framework(s) Used

| Framework | Evidence | Role |
|-----------|----------|------|
| **Vanilla JS** | `extension/sidepanel.js` (74K, no import statements visible) | Chrome extension sidebar UI |
| **Plain HTML/CSS** | `extension/sidepanel.html`, `extension/sidepanel.css` (37K) | Sidebar markup and styling |
| No React/Vue/Svelte | Absence from package.json | Extension runs in Chrome extension context |

The extension UI is vanilla because: Chrome extension content scripts have limited module support; the extension has a very focused interaction surface (chat + command input) that doesn't require a framework's reactivity.

---

## Storage Technologies Used

| Technology | Evidence | Responsibility |
|------------|----------|---------------|
| **File system (JSON)** | `~/.gstack/browse.json`, `~/.gstack/config.yaml` | Server state, user config |
| **File system (JSONL)** | `~/.gstack/security/attempts.jsonl`, `~/.gstack/projects/*/evals/` | Security attack log, eval history |
| **Bun native SQLite** | `package.json` mentions SQLite support (no better-sqlite3 dep visible) | Likely used for tab/session state in browser-manager |
| **Supabase** | `supabase/functions/`, `bin/gstack-telemetry-sync` | Analytics/telemetry backend |
| **Git** | `lib/worktree.ts`, various skill templates | Source control, worktree isolation for tests |

**Storage philosophy**: Local files first, external backend for aggregation only. State stays on the user's machine.

---

## Vector / Memory Technologies Used

| Technology | Evidence | Responsibility |
|------------|----------|---------------|
| **ONNX Runtime** | `@huggingface/transformers` v4, `browse/src/security-classifier.ts` (ONNX) | TestSavantAI ML classifier for prompt injection |
| **File-based learnings** | `bin/gstack-learnings-log`, `scripts/resolvers/learnings.ts` | Cross-session learning storage |
| **No vector DB** | Absence from package.json | Learnings retrieved by recency/relevance via file search, not embeddings |

**Notable**: The ML component (ONNX prompt injection classifier) is used for security classification, not semantic retrieval. This is an unusual and interesting use of ML in an AI agent system.

---

## Queue / Job / Scheduling Technologies Used

| Technology | Evidence | Responsibility |
|------------|----------|---------------|
| **GitHub Actions cron** | `.github/workflows/evals-periodic.yml` → `schedule: cron: '0 6 * * 1'` | Weekly periodic eval runs |
| **No queue system** | Absence of Bull/BullMQ/Celery | Tasks are synchronous or GitHub Actions-scheduled |

No persistent job queue. Long-running tasks (E2E evals) are GitHub Actions workflows with matrix parallelism.

---

## Auth / Session Technologies Used

| Technology | Evidence | Responsibility |
|------------|----------|---------------|
| **Random bearer token** | `~/.gstack/browse.json` token field, `Authorization: Bearer` headers | CLI → daemon auth |
| **Scoped tokens** | `browse/src/cli.ts` pair-agent command | Remote agent restricted surface |
| **HttpOnly SSE cookie** | CLAUDE.md §transport-layer security: `gstack_sse` cookie, `POST /sse-session` | Sidebar SSE session auth |
| **Setup key** | pair-agent generates one-time setup key for initial pairing | Initial token exchange with remote agent |

No OAuth, no JWT, no external auth provider. Token-based with short-lived scoped tokens is appropriate for a local-network tool.

---

## Observability / Logging / Tracing Stack

| Technology | Evidence | Responsibility |
|------------|----------|---------------|
| **Structured JSONL** | `~/.gstack/security/attempts.jsonl` | Security denial logging |
| **Structured JSON** | `~/.gstack/projects/*/evals/*.json` | Eval results + trend data |
| **Console/stdout** | Browse daemon stdout (inferred) | Server logs |
| **Supabase functions** | `bin/gstack-telemetry-sync` | Aggregate telemetry (inferred) |
| **No distributed tracing** | Absence of OpenTelemetry/Jaeger/DataDog | Single-process tool; no trace propagation needed |
| **`bun run eval:compare`** | `scripts/eval-compare.ts` | Manual eval trend comparison |

Observability is primarily local and file-based. The eval store provides the best structured insight into system behavior over time.

---

## Test Frameworks and Test Organization

| Framework | Evidence | Responsibility |
|-----------|----------|---------------|
| **Bun test** (`bun:test`) | `bun test`, all test files import from `bun:test` | Test runner |
| **Custom session-runner** | `test/helpers/session-runner.ts` | E2E tests via `claude -p` subprocess |
| **LLM-as-judge** | `test/helpers/llm-judge.ts` | Automated quality scoring |
| **Custom eval-store** | `test/helpers/eval-store.ts` | Persistent result tracking |
| **Custom touchfiles** | `test/helpers/touchfiles.ts` | Diff-based test selection |

**Test organization**:
```
test/
├── skill-validation.test.ts    — Tier 1: free, <2s, catches template errors
├── gen-skill-docs.test.ts      — Tier 1: generator quality checks
├── skill-llm-eval.test.ts      — Tier 2: LLM judge (~$0.15)
├── skill-e2e-ship.test.ts      — Tier 2 gate: ship workflow E2E
├── skill-e2e-review.test.ts    — Tier 2 gate: review workflow E2E
├── skill-e2e-investigate.test.ts — Tier 2 gate or periodic
├── codex-e2e.test.ts           — Tier 3: periodic (requires Codex auth)
└── helpers/                    — Shared infrastructure
```

---

## Deployment / Runtime Assumptions

| Assumption | Evidence |
|------------|----------|
| Single user per machine | `~/.gstack/` single-user directory structure |
| macOS ARM64 is primary platform | codesign step in setup, binary is Mach-O arm64 |
| Linux supported (CI runs on Ubicloud Linux) | `.github/workflows/evals.yml` |
| Windows supported (with Node.js fallback) | `browse/src/cli.ts` Windows branch |
| Claude Code installed separately | SKILL.md is consumed by Claude Code, not shipped with gstack |
| Bun >= 1.0.0 installed | `package.json` engines |
| Internet access available | Playwright Chromium download, model downloads, Supabase telemetry |
| No container/Kubernetes needed | Local tool, file-system state |
| CI: GitHub Actions on Ubicloud | `.github/workflows/evals.yml` Ubicloud runner |
| CI: Docker image pre-baked | `.github/docker/Dockerfile.ci` with Playwright + Chromium |
