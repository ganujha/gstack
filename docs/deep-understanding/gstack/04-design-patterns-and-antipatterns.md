# 04-design-patterns-and-antipatterns.md — gstack

## Strong Design Patterns

### 1. Persistent Daemon for Sub-Second Tool Use
**Pattern**: Run Chromium as a long-lived HTTP server; CLI connects per command.
**Why strong**: Eliminates 2-3s cold-start overhead per command. 20-command session = ~40s saved. Enables stateful browsing (cookies, localStorage, open tabs persist).
**Transferable**: Any AI agent tool with expensive initialization should use this pattern (database connections, model loading, subprocess boot).

### 2. Prompt Template Compilation (Skill System)
**Pattern**: Skill prompts are compiled from typed TypeScript resolvers into markdown files. Hand-authored `.tmpl` files contain `{{PLACEHOLDERS}}`; a build step expands them.
**Why strong**:
- Type safety at build time (TypeScript resolver functions)
- Shared sections (preamble, review specs) used across skills without duplication
- Host-aware output (same template → 10 different AI agents)
- Freshness enforcement (CI fails if generated SKILL.md is stale)
- Token budget tracking (warns at 160KB)
**Transferable**: Any system with many LLM prompts that share sections benefits from a prompt compilation step.

### 3. Preamble Tier System
**Pattern**: Skills choose a tier (1-4); each tier adds self-regulation behavior. Tier 1 = minimal, Tier 4 = fully orchestrated.
**Why strong**: Avoids the trap of copying-and-pasting preamble content into every skill. Adding a new instruction to all Tier-2+ skills requires one change to one generator file.
**Transferable**: Multi-skill systems should have a "behavioral contract" that skills opt into by tier, not copy by hand.

### 4. Diff-Based Test Selection
**Pattern**: `touchfiles.ts` declares which test depends on which source files. `git diff` determines which tests run. `EVALS_ALL=1` forces all.
**Why strong**: At ~$4/run for gate evals, running all tests on every PR is expensive. Diff-based selection reduces cost while maintaining coverage for affected paths.
**Transferable**: Any test suite with significant per-run cost benefits from dependency-aware test selection.

### 5. LLM-as-Judge Quality Evaluation
**Pattern**: Generated SKILL.md files are scored by a separate LLM (Claude Sonnet) on clarity, completeness, and actionability. Results persist with trend tracking.
**Why strong**: Human review of generated prompts at scale is impractical. LLM judge provides automated quality gates with reproducible scoring rubrics.
**Transferable**: Prompt engineering pipelines should include automated quality scoring with baseline comparison.

### 6. @Ref-Based DOM Element References
**Pattern**: Snapshot assigns `@e1`, `@e2`... refs to interactive elements. Agent uses refs, not XPath or CSS selectors.
**Why strong**:
- Agent never needs to reason about CSS specificity or XPath syntax
- Refs are stable within a snapshot session (locator map is preserved server-side)
- Shorter, more readable in agent context than full selectors
**Risk**: Refs expire when page changes; agent must re-snapshot. This is a correctness requirement baked into skills.

### 7. Dual-Listener Port Separation for Security
**Pattern**: Bind two separate TCP ports (local and tunnel). ngrok forwards only the tunnel port. Security is enforced by physical port separation, not header inspection.
**Why strong**: Header-based origin checks can be spoofed. Physical port separation means the attacker cannot reach bootstrap endpoints even with correct headers.
**Transferable**: Any system exposing a subset of its API surface via a reverse proxy should use port-level separation, not header-level gating.

### 8. Ensemble ML Classifier with Deterministic Canary
**Pattern**: Multiple ML models vote; BLOCK requires 2-of-3 agreement at WARN threshold. Canary injection (L5) is deterministic and always overrides.
**Why strong**: Single ML classifier has FP risk (Stack Overflow instruction writing). Ensemble + deterministic layer prevents both false positives and missed injections.
**Transferable**: AI security systems should always combine probabilistic ML signals with deterministic tripwires.

### 9. Evidence-Before-Action Protocol (investigate skill)
**Pattern**: The `/investigate` skill has an Iron Law: no fixes without root cause investigation first. 3 phases before touching code.
**Why strong**: Prevents the common failure mode where AI agents apply surface-level fixes that mask root causes. The skill enforces this structurally via a freeze state file (`~/.gstack/freeze-dir.txt`).

### 10. CHANGELOG as User-Facing Product Release Notes
**Pattern**: Every version entry starts with a "release summary" in plain language (what users can now do), followed by itemized changes. Internal/contributor details go at the bottom.
**Why strong**: Forces writers to think from user perspective. Release notes that lead with "refactored" don't tell users anything.

---

## Weak Patterns / Risks / Technical Debt

### 1. Binary Files Tracked in Git
**Risk**: `browse/dist/` and `design/dist/` are committed binaries (58MB each, Mach-O arm64). They silently fail on Linux, Windows, Intel Macs.
**Why weak**: `git add .` accidentally includes them. CI on Linux builds from source anyway — the tracked binaries serve no purpose.
**Fix**: `git rm --cached browse/dist/ design/dist/` (acknowledged in CLAUDE.md as "historical mistake").

### 2. Large SKILL.md Files in Context
**Risk**: `ship/SKILL.md` is 42K tokens; `review/SKILL.md` is likely similar. These occupy 20-40% of a 200K context window before any user content is loaded.
**Consequence**: Complex PRs with large diffs may hit context limits mid-workflow. The 160KB warning exists but is advisory.
**Mitigation in place**: Prompt caching makes repeated context cheaper; preamble is cached across turns.

### 3. Single Chromium Instance Per User
**Risk**: Parallel skill executions (two Claude Code windows) share one browser. Tab switching logic and state may interfere.
**Mitigation**: `BROWSE_TAB` env var and `--tab-id` flag scope commands to specific tabs. But no locking at the browser level.

### 4. Resolver File Size Sprawl
**Risk**: `review.ts` (54K), `design.ts` (58K), `testing.ts` (27K), `utility.ts` (17K). These are gigantic single files generating prompt text.
**Consequence**: Hard to navigate, easy for unrelated reviewer concerns to mix. The 160KB token ceiling guards against output size, but not source file complexity.

### 5. Inferred `[[PLACEHOLDER]]` Errors at Runtime
**Risk**: If a resolver function throws or returns empty, the placeholder may appear literally in the generated SKILL.md (e.g., `{{REVIEW_ARMY}}`). Claude would then act on the literal placeholder text.
**Mitigation in place**: `skill-validation.test.ts` catches stale/missing placeholders in free tests. But dynamic resolver failures (those depending on external state) may not be caught until runtime.

### 6. Windows Support as Second-Class
**Risk**: Windows requires a different server entrypoint (`server-node.mjs`), different binary compilation path, longer health check timeout (15s vs 8s), `taskkill /T /F` instead of `kill -9`.
**Consequence**: Windows-specific bugs are harder to reproduce in CI (CI runs on Linux).

### 7. No E2E Test Coverage for Security Layers
**Risk**: The 6-layer security stack (`security.ts`, `content-security.ts`, `security-classifier.ts`) is documented but the extent of automated testing is unclear from available evidence.
**Uncertainty**: May have dedicated security tests not surfaced in the touchfiles analysis.

---

## Hidden Assumptions

1. **One user per machine**: `~/.gstack/` uses a single directory for all state. Two users on the same machine would collide.
2. **Claude Code is the primary runtime**: Skills reference `$B` alias, Claude-specific tools (`AskUserQuestion`, `TodoWrite`), and Claude Code's permission model. Skills generated for Codex, Factory, etc. require path rewrites and tool renames — the default is always Claude.
3. **Bun is the build + runtime**: No fallback build path for npm/yarn. Only Windows gets a Node.js server fallback.
4. **User has a GitHub remote**: Skills like `/ship` assume `gh pr create` is available and the repo has a remote. The `gstack-repo-mode` utility handles local-only repos, but it's opt-in.
5. **Internet access for tunnels**: `pair-agent` assumes ngrok is available. No fallback for air-gapped environments.
6. **ML model download succeeds**: First run of TestSavantAI classifier downloads 112MB to `~/.gstack/models/`. No offline fallback.

---

## Likely Failure Modes

### Availability
- **Browse server crash mid-workflow**: One retry on ECONNREFUSED, then failure. Long skill workflows (ship, autoplan) have no checkpointing against server crashes.
- **ML model download failure**: First-run security classifier setup silently degrades (GSTACK_SECURITY_OFF behavior, inferred). No user notification.
- **ngrok tunnel disconnection**: `pair-agent` mode loses remote agent access; no reconnect logic visible.

### Correctness
- **Stale @refs**: Agent uses `@e3` from a snapshot but page has changed; click fails silently or clicks wrong element. No automatic re-snapshot before write commands.
- **Resolver produces wrong content for host**: A placeholder resolver that assumes Claude tool names may generate invalid content for Codex/Factory hosts if path rewrite misses a case.
- **Concurrent skill execution**: Two Claude Code instances writing to the same TODOS.md, CHANGELOG.md, VERSION simultaneously — git conflict, not an atomic operation.

### Context management
- **Context window exhaustion**: Large SKILL.md + large diff + long tool history → context truncation → Claude loses awareness of earlier steps. `generate-continuous-checkpoint.ts` mitigates but doesn't prevent.
- **Confusion spiral**: Claude asks for clarification, user responds ambiguously, Claude asks again. No termination condition in the confusion protocol.

---

## Reliability Risks

- **Atomic file operations**: `acquireServerLock()` uses `wx` flag (atomic on POSIX). But state file writes (`browse.json`) may not be atomic (depends on Bun's write semantics).
- **Security log rotation**: Attack log rotates at 10MB across 5 generations. If filesystem is full, rotation fails silently (async `appendFile` with no error handler visible in evidence).
- **Eval cost accumulation**: No per-PR spend limit enforced. At ~$8 total/PR (gate + E2E), high-velocity PRs could accumulate significant API spend.

---

## Context-Management Risks

- In-context state (TodoWrite) is lost if the Claude Code process restarts
- No automatic snapshot of context before a skill's long workflow begins
- `generate-context-recovery.ts` is a prompt instruction, not a code enforcement — Claude may ignore it under heavy context pressure

---

## Tool-Selection Risks

- `$B` commands require the browse daemon to be running; if the server isn't started and the health check fails repeatedly, the entire skill workflow fails
- Agent tool for sub-review spawn has no timeout enforcement — if a sub-reviewer hangs, the coordinator waits indefinitely

---

## Security / Safety Risks

- **Ensemble ML is stochastic**: Two independent classifiers may both mislabel benign content as injection and BLOCK a legitimate page. No user-override path visible (besides `GSTACK_SECURITY_OFF`).
- **Datamarking detection only at write time**: If an injection exfiltrates data before the canary check runs, it's logged but not prevented.
- **Codex trust boundary**: The skill template instructs Codex not to read `~/.claude/` — but this is a prompt instruction, not an OS-level permission. A malicious Codex prompt override could circumvent it.

---

## Observability Gaps

- No distributed tracing across multi-agent skill executions
- No structured logging from the browse daemon (inferred: stdout/stderr only, no log aggregation)
- Eval results stored locally; no centralized dashboard visible
- Security attack log is per-user local; no fleet-wide aggregation
- `gstack-specialist-stats`, `gstack-telemetry-log`, `gstack-analytics` suggest some observability — but exact output format and collection pipeline is unclear

---

## Cost / Latency Trade-offs

| Decision | Cost | Latency | Quality |
|----------|------|---------|---------|
| Persistent Chromium daemon | Memory (~200MB) | Sub-second commands | High |
| Large SKILL.md in context | Token cost per turn | Slow first turn (cache miss) | High |
| LLM-as-judge evals | ~$0.15/eval run | 30-60s | Automated quality |
| E2E via claude -p | ~$3.85/run | 30-45 min | True end-to-end |
| ML classifiers (ONNX) | Disk (112MB) + CPU | ~10-50ms inference | Prompt injection defense |
| Ensemble + transcript | API cost (Haiku) | +200ms per page | Lower FP rate |

---

## Suggestions for Hardening

1. **Atomic state file writes**: Use `writeFileSync` to `.tmp` → `rename` for `browse.json` (prevents partial reads during concurrent access)
2. **Browse daemon restart checkpointing**: Long skills should write last-completed-step to disk; on server restart, resume from checkpoint
3. **Resolver output validation**: Add a post-generation pass that checks for literal `{{PLACEHOLDER}}` strings in output SKILL.md files
4. **Windows CI lane**: Add a Windows CI runner for at least smoke-testing the Node.js fallback path
5. **Concurrent write protection**: Skills that write TODOS.md/CHANGELOG.md should use git lock or file lock to prevent two simultaneous skill sessions corrupting files
6. **Security test coverage**: Add E2E tests that inject known attack payloads and verify they're blocked at the correct layer
7. **Spend limit enforcement**: `touchfiles.ts` could add a `maxCostCents` per test suite that's enforced before execution
