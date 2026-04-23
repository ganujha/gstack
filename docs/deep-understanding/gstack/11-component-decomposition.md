# 11-component-decomposition.md — gstack

## Infrastructure / Runtime Layer

This layer provides the execution environment and hardware-adjacent services.

### Components

| Component | Custom vs. Third-Party | Central or Replaceable |
|-----------|----------------------|----------------------|
| **Bun runtime** | Third-party (Oven) | Central (entire build/run pipeline) |
| **Playwright** | Third-party (Microsoft) | Replaceable (Puppeteer alternative exists) |
| **Chromium** | Third-party (Google) | Partially replaceable (Firefox via Playwright) |
| **ONNX Runtime** (via @huggingface/transformers) | Third-party | Replaceable (other ML inference engines) |
| **ngrok** | Third-party (optional) | Replaceable (any tunneling service) |
| **Supabase** | Third-party | Replaceable (any PostgreSQL-compatible backend) |
| **GitHub Actions** | Third-party | Replaceable (any CI) |
| **Ubicloud** | Third-party (CI runner) | Replaceable (any Linux runner) |

### How Components Interact
- Bun provides TypeScript execution and HTTP server (Bun.serve) for the browse daemon
- Playwright controls Chromium via CDP (Chrome DevTools Protocol)
- ONNX Runtime loads TestSavantAI model for ML inference (in sidebar-agent only)
- ngrok creates public tunnel to the local browse daemon tunnel port

### What Depends on What
```
User CLI ($B) → Browse binary (Bun compiled) → Bun.serve HTTP server → Playwright → Chromium
Test runner (bun test) → session-runner → claude binary → Claude Code → SKILL.md
```

---

## Framework Layer

This layer provides reusable abstractions that application code builds on.

### Components

| Component | Location | Custom vs. Third-Party | Role |
|-----------|----------|----------------------|------|
| **Bun.serve** | Runtime | Third-party | HTTP routing in browse/src/server.ts |
| **Playwright API** | Runtime | Third-party | Browser automation primitives |
| **Anthropic SDK** | `@anthropic-ai/sdk` | Third-party | LLM API calls (judge, sidebar) |
| **bun:test** | Runtime | Third-party | Test framework |
| **diff** | Third-party | Utility | Snapshot diff computation |
| **marked** | Third-party | Utility | Markdown rendering |

### What is Custom Framework-Like Code
- **HostConfig system** (`scripts/host-config.ts`) — typed interface + validation for skill distribution
- **Resolver pattern** (`scripts/resolvers/index.ts`) — `Record<string, ResolverFn>` registry for template expansion
- **Preamble tier system** (`scripts/resolvers/preamble.ts`) — composable behavioral contract assembly
- **Session runner** (`test/helpers/session-runner.ts`) — subprocess-based E2E test framework

These are internal frameworks: stable abstractions that all higher-level code depends on.

---

## Application / Service Layer

The primary deliverables: the browse daemon, skill system, and CLI utilities.

### Browse Daemon Service

```
browse/src/
├── cli.ts          — Entry: command dispatch, server lifecycle, pair-agent
├── server.ts       — Core: HTTP routing, Chromium management, security orchestration
├── browser-manager.ts — State: tab management, Playwright page lifecycle
├── snapshot.ts     — Feature: ARIA tree → @ref-annotated text
├── commands.ts     — Catalog: 50 commands (read/write/meta)
├── read-commands.ts  — Implementation: text, html, links, forms, etc.
├── write-commands.ts — Implementation: goto, click, fill, type, etc.
├── meta-commands.ts  — Implementation: screenshot, pdf, diff, watch, etc.
└── error-handling.ts — Utility: safeUnlink, safeKill, isProcessAlive
```

**Custom**: Entire browse daemon is custom code on top of Playwright.
**Architectural core**: `server.ts` + `cli.ts` — replacing these would require rewriting the product.
**Replaceable**: Individual command implementations (e.g., `read-commands.ts`) could be swapped without changing the architecture.

### Skill Generation Service

```
scripts/
├── gen-skill-docs.ts         — Core: template processing pipeline
├── resolvers/index.ts        — Core: placeholder → resolver mapping
├── resolvers/preamble.ts     — Core: tier-based preamble assembly
├── resolvers/preamble/*.ts   — Content: 22 behavioral instruction generators
├── resolvers/review.ts       — Content: CEO/Eng/Design/DX review specs (54K)
├── resolvers/design.ts       — Content: design audit checklist (58K)
├── resolvers/testing.ts      — Content: test framework detection (27K)
├── resolvers/utility.ts      — Content: shared utility sections (17K)
├── resolvers/browse.ts       — Content: browse daemon documentation (6K)
└── resolvers/review-army.ts  — Content: multi-reviewer dispatch (11K)
```

**Custom**: The entire resolver system is custom.
**Architectural core**: `gen-skill-docs.ts` + `resolvers/index.ts` + `resolvers/preamble.ts`.
**Replaceable**: Individual resolver content modules can be modified independently.

### CLI Utilities (bin/)

38 shell/TypeScript scripts providing per-feature functionality:
- Config management (`gstack-config`)
- Telemetry (`gstack-telemetry-log`, `gstack-telemetry-sync`)
- Eval management (`gstack-learnings-log`, `gstack-learnings-search`)
- Development tools (`gstack-diff-scope`, `gstack-repo-mode`)
- User profiling (`gstack-developer-profile`, `gstack-question-preference`)

**Replaceable**: Each utility is standalone. Adding or removing one doesn't affect others.

### Chrome Extension Service

```
extension/
├── manifest.json   — Extension manifest (MV3)
├── background.js   — Service worker (tab management, auth)
├── content.js      — Page interaction (injected into pages)
├── sidepanel.js    — Sidebar chat UI (74K, most complex)
├── sidepanel.html  — Sidebar markup
├── inspector.js    — DOM inspector
└── popup.html/.js  — Extension popup
```

**Custom**: Entire extension is custom.
**Architectural core**: `sidepanel.js` + `background.js` (they implement the chat protocol).
**Replaceable**: The extension is an optional headed-mode component; core browse daemon works without it.

---

## Orchestration Layer

This layer coordinates multi-step workflows.

### Skill Templates (Skills-as-Orchestration)

Each SKILL.md is itself an orchestration specification:
- **`ship/SKILL.md`**: 19-step shipping pipeline (git merge → test → version bump → CHANGELOG → push → PR → canary)
- **`autoplan/SKILL.md`**: CEO → Design → Eng → DX sequential review pipeline
- **`review/SKILL.md`**: Critical pass → specialist fanout → fix-first → ask gate

**Important**: Orchestration happens inside Claude's language model, not in code. The skill template specifies the workflow; Claude executes it using its built-in agent loop and tool calls. There is no workflow engine, no state machine, no explicit orchestrator code.

### Agent Fanout (review-army.ts resolver)

`scripts/resolvers/review-army.ts` generates prompt instructions for parallel specialist dispatch. This is orchestration-via-prompt: Claude reads instructions telling it to spawn N subagents for N review domains, then aggregate results.

### Test Orchestration (GitHub Actions)

`.github/workflows/evals.yml` runs 12 test suites in matrix parallelism across Ubicloud runners. This is traditional CI orchestration, not agent orchestration.

---

## Memory / Context Layer

This layer handles information persistence and retrieval.

### Components

| Component | Location | Type | Scope |
|-----------|----------|------|-------|
| **In-context scratchpad** | Claude context window | Ephemeral | Session |
| **TodoWrite** | Claude context + tool | Ephemeral | Session |
| **Server state** | `~/.gstack/browse.json` | JSON file | Per-daemon-lifetime |
| **User config** | `~/.gstack/config.yaml` | YAML file | Persistent |
| **Security state** | `~/.gstack/security/session-state.json` | JSON file | Per-session (shared across processes) |
| **Security attack log** | `~/.gstack/security/attempts.jsonl` | JSONL | Persistent (10MB rotated) |
| **Eval store** | `~/.gstack/projects/$SLUG/evals/` | JSON files | Persistent |
| **Learnings** | `~/.gstack/learnings/` (inferred) | Files | Persistent |
| **ML model cache** | `~/.gstack/models/` | Binary files | Persistent |
| **Skill content** | `~/.claude/skills/gstack/*/SKILL.md` | Markdown | Persistent (updated on upgrade) |
| **Git state** | `.git/` | Git objects | Persistent |

### How Memory Components Interact
- Skills read file-system memory (TODOS.md, CHANGELOG.md) into in-context memory
- Skills write in-context decisions to file-system memory (UPDATE TODOS.md)
- Security state is shared between `server.ts` and `sidebar-agent.ts` via JSON file
- Eval store namespaces by project via `gstack-slug`

---

## Interface Layer

This layer is the user-facing surface.

### Components

| Interface | Type | Users |
|-----------|------|-------|
| **Slash commands** (`/ship`, `/review`) | Claude Code UX | Primary users |
| **`$B` CLI** | Shell command | Skills (and users directly) |
| **Chrome extension sidebar** | Browser UI | Headed-mode users |
| **`./setup`** | Shell script | Installation (one-time) |
| **`bun run *` scripts** | npm-style scripts | Developers/contributors |
| **`bin/gstack-*`** | Shell utilities | Advanced users |
| **GitHub Actions** | CI triggers | Contributors |

### Key Interface Design Choices
- `$B` is a short alias by design: minimizes tokens used in skill templates
- @refs abstract away CSS/XPath: reduces agent error surface
- Slash commands are the product: everything else is infrastructure
- Setup is one-command: `./setup` handles everything (build, link, migrate)

---

## Cross-Cutting Concerns

### Security
- **Prompt injection defense**: 6 layers (content-security.ts, security.ts, security-classifier.ts)
- **Transport security**: dual-listener topology, bearer tokens, scoped tokens, HttpOnly cookies
- **Data minimization**: attack log stores salted sha256 + domain only (not full URLs)
- **Audit**: `~/.gstack/security/attempts.jsonl` for tunnel denials
- **Kill switch**: `GSTACK_SECURITY_OFF=1`

**What's custom**: The entire security stack is custom-built. No third-party WAF or API gateway.

### Logging / Observability
- Browse daemon: stdout (no structured logging framework)
- Security: JSONL append (custom)
- Evals: JSON store (custom)
- Telemetry: Supabase (inferred, custom pipeline)

**Gap**: No distributed tracing, no log aggregation, no APM.

### Configuration
- **Build-time**: `hosts/*.ts` (TypeScript, type-checked)
- **Runtime**: `~/.gstack/config.yaml` (YAML, read by `bin/gstack-config`)
- **Env vars**: `ANTHROPIC_API_KEY`, `GSTACK_SECURITY_OFF`, `EVALS_ALL`, `EVALS_TIER`

### Error Handling
- `browse/src/error-handling.ts`: `safeUnlink`, `safeUnlinkQuiet`, `safeKill`, `isProcessAlive`
- Extension: catch-and-log (Chrome extensions crash on uncaught errors)
- Cleanup paths: `safeUnlinkQuiet` swallows all errors (ensures cleanup completes)
- Normal paths: `safeUnlink` ignores ENOENT, rethrows EPERM/EIO

### Testing
- Skill validation: static analysis (free, <2s)
- LLM judge: automated quality scoring (~$0.15/run)
- E2E: `claude -p` subprocess execution (~$3.85/run)
- Coverage: diff-based selection via touchfiles.ts

### Version Management
- `VERSION` file: current version string
- `CHANGELOG.md`: full release history (321K)
- `~/.gstack/.last-setup-version`: tracks installed version for migration gating
- `gstack-upgrade/migrations/v*.sh`: idempotent version-specific migrations

---

## What Is Architectural Core vs. Replaceable

### Architectural Core (replacing = rewriting the product)
- Browse daemon persistent model (cli.ts + server.ts)
- Skill-as-prompt compilation pipeline (gen-skill-docs.ts + resolvers/index.ts)
- Preamble tier system (preamble.ts)
- HostConfig abstraction (host-config.ts + hosts/index.ts)
- Diff-based test selection (touchfiles.ts)
- Security ensemble model (security.ts combineVerdict)

### Replaceable with Limited Impact
- Specific resolver content modules (review.ts, design.ts) — content can change without architecture change
- Individual browse commands (read-commands.ts, write-commands.ts) — new commands without CLI change
- Chrome extension (optional headed mode component)
- Supabase backend (any PostgreSQL-compatible backend works)
- Playwright → Puppeteer (same CDP protocol, similar API surface)
- ngrok → any tunneling service
- Bun.serve → any HTTP server (larger change, but architecturally isolated to server.ts)
- Individual bin/ utilities (each is standalone)
