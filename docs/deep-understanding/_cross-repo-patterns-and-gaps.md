# _cross-repo-patterns-and-gaps.md

> Cross-subsystem synthesis across gstack's internal subsystems (browse daemon, skill compile, security stack, test infrastructure, Chrome extension). This doc answers: what patterns recur and are worth stealing? What weaknesses recur and need fixing? Where does one subsystem solve elegantly what others stumble on?

---

## Best Patterns Worth Reusing

### P1. Typed static registry + fail-fast expansion (skill compile)
`Record<string, ResolverFn>` registry, single-pass regex expansion, fail-fast on any unresolved placeholder at build time.

**Why it's good**: zero runtime surprises, complete type coverage, trivial to debug.
**Where it's used**: `scripts/resolvers/index.ts`.
**Where to steal it**: any prompt/template/config pipeline with multiple output targets.

### P2. Dual-listener physical port separation (browse daemon)
Two `Bun.serve` calls on different ports. One is local-only, one is tunnel-exposed. Allowlists (path + command) enforced at route dispatch, not via headers.

**Why it's good**: headers lie; ports don't. Upstream proxies can forge origin; they can't pick a port you didn't bind.
**Where it's used**: `browse/src/server.ts:2712, 2785`.
**Where to steal it**: any local service exposing a subset of its API via a reverse proxy or tunnel.

### P3. Ensemble ML + deterministic canary (security stack)
Multiple stochastic signals, plus one deterministic tripwire. `combineVerdict` requires 2-of-N agreement at WARN threshold before escalating to BLOCK.

**Why it's good**: single-classifier false positives die; deterministic layer catches what ML misses.
**Where it's used**: `browse/src/security.ts:96-178`.
**Where to steal it**: any AI safety system that can't tolerate single-classifier FPs but can afford ensemble inference cost.

### P4. Cost-aware routing by intent regex (sidebar)
15-line regex decides between Opus (analysis) and Sonnet (actions). Cheap path for structural requests; expensive path for ambiguous ones.

**Why it's good**: zero inference cost, deterministic, adequate accuracy for the intent split.
**Where it's used**: `browse/src/server.ts:190-211`.
**Where to steal it**: anywhere you're about to train an ML classifier for a 2-class routing problem with a clean keyword signature.

### P5. Diff-based per-test dependency declaration (test infrastructure)
`touchfiles.ts` maps tests to source files. `git diff` decides what runs. Global touchfiles trigger everything.

**Why it's good**: paid tests cost less; unchanged code doesn't re-verify.
**Where it's used**: `test/helpers/touchfiles.ts`.
**Where to steal it**: any test suite with per-run cost (end-to-end tests, integration tests, LLM-as-judge evals).

### P6. Atomic state writes via .tmp + rename (browse daemon)
Write to `.tmp`, then `fs.renameSync`. Guaranteed atomicity on POSIX; no partial-read window.

**Why it's good**: no corruption during concurrent access; no partial reads.
**Where it's used**: `browse/src/server.ts:2728-2730`.
**Where to steal it**: any state file written by a long-running process that might be read concurrently.

### P7. Startup lock via `wx` atomic create (browse daemon)
`fs.openSync(lockPath, 'wx')` atomically succeeds only if the file doesn't exist. Releases in try/finally.

**Why it's good**: free (kernel-level), atomic, cross-platform.
**Where it's used**: `browse/src/cli.ts:283-306`.
**Where to steal it**: any service where concurrent startup would race.

### P8. Per-tier additive composition of behavioral contracts (skill compile)
Preamble tiers (1-4) compose additively: each tier adds sections, doesn't replace. Skills opt in by tier number.

**Why it's good**: adds behavior to all skills of a given tier with one file change. Skills inherit by contract, not by copy.
**Where it's used**: `scripts/resolvers/preamble.ts:71-103`.
**Where to steal it**: any prompt framework with 20+ skills sharing behavioral norms.

### P9. Server-side reference map (@refs) for agent-facing locators (browse daemon)
Agent uses `@e3`, server maintains the ref→Playwright locator. Agent never sees CSS/XPath.

**Why it's good**: eliminates a whole class of agent errors (selector syntax); shortens tokens.
**Where it's used**: `browse/src/snapshot.ts`, `browse/src/browser-manager.ts`.
**Where to steal it**: any agent interface where the agent needs to address entities that have complex underlying identifiers.

### P10. Dynamic per-run cost accounting (test infrastructure)
`pricing.ts` computes cost from actual `resultLine.usage` × model-specific rate tables.

**Why it's good**: costs are real, not budgeted. Dashboards can show true spend.
**Where it's used**: `test/helpers/pricing.ts`.
**Where to steal it**: any paid-LLM workflow where cost visibility matters.

---

## Recurring Weaknesses Across Subsystems

### W1. Documentation-enforced load-bearing invariants
Appears in **security stack** (`security-classifier.ts` must not be imported from `server.ts`), **skill compile** (no mechanical import-boundary checks for host configs), **browse daemon** (`token-registry.ts` must not be imported from `sse-session-cookie.ts`). Doc rules; no lint.

### W2. Stochastic systems treated as comparable
Appears in **test infrastructure** (LLM judge has no temperature pinning; regex JSON extraction; results compared across runs) and implicitly in **security stack** (ML classifier scores have variance that threshold tuning assumes away).

### W3. No cost caps at any layer
**Test infrastructure** has no per-PR spend limit. **Security stack** has no API quota for the Haiku transcript classifier. **Browse daemon** doesn't rate-limit `$B` calls. Every path assumes the caller is sensible.

### W4. Prompt-only trust boundaries
**Security stack**: Codex "don't read `~/.claude/`" is a prompt instruction, not an OS permission. **Skill compile**: the "don't auto-accept community PRs touching ETHOS.md" rule is a comment in CLAUDE.md, not a CI gate. **Test infrastructure**: "don't claim pre-existing failure" is instructed, not enforced.

### W5. Single-user, single-Chromium concurrency assumptions
**Browse daemon**: one Chromium per user, tab state shared across concurrent skill sessions. **Chrome extension**: one sidepanel per browser window.

### W6. No checkpoint/resume in long workflows
**Browse daemon idle timer** can kill the daemon mid-skill. **Test infrastructure `claude -p` sessions** fail entirely on turn-limit exhaustion. **Skill compile** is fast enough this doesn't matter.

---

## Surprising Differences Between Subsystems

### D1. Browse daemon is the only subsystem without decisions in its code
Every other subsystem either compiles decisions (skill compile), classifies them (security), selects them (test infrastructure), or routes them (extension sidebar). Browse daemon is deliberately "dumb I/O." This is a purity worth noting.

### D2. Skill compile has zero runtime state; security stack has five state files
The most stateful subsystem (security: session state, attack log, device salt, model cache, classifier cache) contrasts with the least stateful (compile: none). Production-readiness correlates with this — more state means more failure modes.

### D3. Test infrastructure has the richest observability; daemon has the weakest
The eval store tracks per-run, per-model, per-tool cost and duration; the daemon logs to stdout. If test-infra observability were generalized to the daemon, production-readiness would jump.

### D4. Security stack accepts stochasticity; tests don't know they're accepting it
Security explicitly designs for stochastic inputs (ensemble, thresholds). Tests assume judge scores are comparable and do not design for variance. Same underlying phenomenon; opposite handling.

### D5. Browse daemon enforces policy at the wire; skills enforce policy in the model
A malformed HTTP request is rejected at `server.ts:1568-1590` before any handler sees it. A malformed user request in a skill goes to the model to interpret. Two legitimate design choices for different surfaces.

---

## What One Subsystem Solves That Others Don't

### Skill compile's fail-fast + CI freshness model
Every subsystem should have a "is this out of sync?" gate. Only skill compile does. Browse daemon can drift from its docs; security stack can drift from its documented invariants; tests can drift from touchfile declarations — none of these have a CI check that proves they're fresh.

### Browse daemon's atomic state + lock pattern
Every subsystem with on-disk state should use this pattern. Only the daemon does. Attack log uses appendFile (not atomic); eval store files are individual writes (not transactional across runs); session-state.json is read/written by two processes without locking.

### Security stack's ensemble-with-deterministic-tripwire
Every probabilistic decision should have a deterministic override for known-bad cases. Only security does it. Skill compile could have hard-coded "always flag this pattern" checks; tests could have "always run this canary test"; neither does.

### Test infrastructure's per-run cost accounting
Every paid LLM interaction across the repo should be priced. Only tests do. The sidebar calls Anthropic per conversation turn with no cost tracking; the judge is costed, but the sidebar is not. Generalizing pricing.ts to all LLM paths would add observability for free.

### Chrome extension's session-cookie SSE auth
Only the extension has an example of rotating session credentials (gstack_sse 30-min HttpOnly cookie). Bearer tokens in `browse.json` are long-lived. Applying the extension's pattern would tighten daemon auth.

---

## What None of the Subsystems Handles Well

### N1. Fleet / multi-user deployment
Every subsystem assumes a single user per machine. `~/.gstack/` is single-rooted; state files are unnamespaced. No subsystem has multi-user awareness.

### N2. Graceful degradation on partial failure
If the Haiku transcript classifier times out, the ensemble rule silently runs with one fewer signal; if Supabase telemetry is down, events are dropped. Nothing surfaces degraded mode to the user.

### N3. Cross-process tracing
The security stack's two-process architecture is elegant but opaque. There's no trace ID that connects a server.ts request to the sidebar-agent.ts decision it depended on. Same for tests that spawn subprocesses.

### N4. Per-skill cost budget
Tests are costed; skills aren't. A `/ship` that burns $5 in sidebar calls is indistinguishable from one that burns $0.50 until the Anthropic invoice arrives.

### N5. Centralized attack analytics
Each user's `attempts.jsonl` is local. No fleet-wide view of "which attack classes are trending." Security research doesn't accumulate across users.

### N6. Automated red-team testing
The security stack is the most sophisticated subsystem but has the least automated adversarial testing. Stack Overflow false positives were fixed, but a regression is only caught by users reporting it.

### N7. Skill template quality enforcement
Skill templates can introduce voice drift, scope drift, or accidental tool reliance. The pipeline checks structural freshness; it doesn't check semantic quality.

### N8. Cross-skill state coordination
Two `/ship` invocations on the same repo will race on VERSION and CHANGELOG. No lock. No detection.

---

## What a Stronger "Combined Architecture" Would Borrow From Each

If you were designing a next-generation version of gstack taking the best of each subsystem:

### From skill compile
- Compile step + typed registry + fail-fast.
- Per-host frontmatter transforms.
- CI freshness gate applied to ALL configuration-like assets (host configs, touchfiles, security thresholds).

### From browse daemon
- Persistent tool-server pattern for anything with expensive initialization.
- Dual-listener topology for any internal-plus-tunnel deployment.
- Atomic state writes + startup locks as default for any on-disk state.
- Bearer-token auth as default between process boundaries.

### From security stack
- Ensemble + deterministic tripwire for any probabilistic safety decision.
- Two-process architectural split where one process has constraints (compiled, size-bounded, no native deps) the other doesn't.
- Salted + minimized attack logs as the default audit model.

### From test infrastructure
- Dynamic per-run cost accounting everywhere paid LLMs are invoked.
- Diff-based selection with touchfiles for any expensive verification.
- Worktree isolation for any test that mutates repo state.
- Structured result store with trend comparison baked in.

### From Chrome extension
- Session-cookie rotation for long-lived browser sessions.
- SSE streaming with token minting separate from the main auth token.

### New elements (none of the above provides, but should)
- **Mechanical enforcement of module boundaries** (ESLint rule or dependency-cruiser) for load-bearing invariants.
- **Per-process trace IDs** threaded through all cross-process calls.
- **Budget daemon** that aggregates spend across skill invocations and halts when over threshold.
- **Signed skill releases** verified at `./setup` time.
- **Fleet telemetry rollup** for attack analytics and skill flakiness.

---

## Summary

gstack's subsystems are **philosophically coherent** (compile, separate, enforce-at-transport) but **not feature-uniform** (only some subsystems have cost accounting, only some have CI freshness gates, only some have atomic writes). The highest-leverage architectural work is generalizing the best patterns from each subsystem to the others.

A single well-chosen generalization — for example, applying skill-compile's CI freshness check to the security module boundaries — would close multiple second-pass risks at once. The repo is mature enough that this kind of cross-subsystem leveling is the natural next phase.
