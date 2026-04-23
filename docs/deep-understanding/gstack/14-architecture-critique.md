# 14-architecture-critique.md — gstack

> A deliberately opinionated review. Strengths where earned, weaknesses where specific. Generic praise and generic advice are excluded — every observation references a concrete file, invariant, or workflow.

---

## Architectural Strengths

### S1. Policy-as-prompt, mechanism-as-code
The skill system forces a clean split: all decisions live in markdown templates; all mechanisms (HTTP, Playwright, SQLite, atomic writes) live in TypeScript. A staff engineer reading `ship/SKILL.md.tmpl` gets the full product specification for shipping. The code directory is small because the code is *only* I/O.

This is durable. Prompts will evolve as models improve; the mechanism doesn't need to. (See `browse/src/server.ts` — 2,835 lines covering HTTP, security, tabs, tunneling, and sidebar chat is the entire server.)

### S2. A compile step for prompts
`gen-skill-docs.ts` is a TypeScript-to-markdown compiler with:
- typed resolver functions (`Record<string, ResolverFn>`)
- static placeholder detection via regex
- fail-fast on missing placeholders at build time
- host-aware frontmatter transforms
- freshness check in CI

Compile steps for prompts are rare in the wild; this one is exemplary in how small it is (≈600 lines of generator, ≈2,000 lines of resolvers) relative to what it outputs (40+ skills × 10 hosts).

### S3. Physical port separation over header-based origin checks
`browse/src/server.ts` binds two `Bun.serve` listeners on different ports. ngrok forwards only the tunnel port. The "allowlist" isn't a CORS header — it's a path set and a command set, both enforced before any handler touches the request. Headers lie; ports don't.

### S4. Ensemble ML + deterministic canary
`combineVerdict` (security.ts:96-178) requires 2-of-N ML classifiers at WARN before escalating to BLOCK, but the canary check in L5 always BLOCKs deterministically when triggered. This kills the "single stochastic classifier" class of false positives (Stack Overflow prose gets flagged as injection) without giving up detection on the actual attack signal (canary string exfiltrated).

### S5. Sidebar model routing by regex
`pickSidebarModel` is 15 lines of regex. It routes messages to Sonnet or Opus based on whether the verb is an imperative action or an analysis question. A typical agent system would invoke a classifier here; gstack does the right thing by not paying for one. The comment captures the reasoning: "sonnet vs opus on 'click @e24' is 5-10x in latency and cost, with zero quality difference." This pattern is worth borrowing.

### S6. Dynamic cost accounting
`test/helpers/pricing.ts` computes actual cost per run from `resultLine.usage` using model-specific pricing tables, rather than a fixed per-test budget. This means eval cost scales with the work each test actually does, and the eval store records real dollars per run for trend analysis.

### S7. Atomic state writes with explicit permissions
Every state file is written as `.tmp → rename`, with `0o600` mode on the browse state file. `acquireServerLock` uses the `wx` flag (atomic exclusive create). These are correct engineering choices with a defensible scope — startup-time only, not steady state.

### S8. Module boundary invariants for architectural isolation
`security-classifier.ts` cannot be imported from the compiled browse binary because `@huggingface/transformers` can't `dlopen` from Bun's temp extract dir. The team didn't fight the tool — they split the process. `server.ts` (compiled, pure-string security) and `sidebar-agent.ts` (non-compiled, ML-heavy) communicate via a session-state JSON file. This is an elegant forced-simplicity architecture.

### S9. Preamble tiers as behavioral contracts
Instead of copy-pasting confusion-protocol instructions into 40 skills, the preamble composes per-tier. Adding a new Tier-2 behavior is one file change. This is composition-over-inheritance done right.

### S10. `@ref` abstraction for selector-less browsing
Hiding CSS/XPath behind short server-side refs reduces agent token use and eliminates a whole class of selector-syntax errors. This is probably gstack's single biggest agent-UX innovation and worth a case study.

---

## Architectural Weaknesses

### W1. Load-bearing invariants are only documentation-enforced
The invariant "`security-classifier.ts` must never be imported by `server.ts`" exists because `@huggingface/transformers` can't `dlopen` from a compiled Bun binary. If a developer imports it by accident, the daemon silently fails at startup on user machines, not in CI. There's no ESLint rule, no dependency-cruiser config, no import boundary test. Same issue for `token-registry.ts ⊄ sse-session-cookie.ts`. These are real, load-bearing module boundaries with zero mechanical enforcement.

**Fix**: add a dependency-cruiser config or a simple `bun test` check that greps source files for forbidden imports.

### W2. The LLM judge is stochastic but treated as comparable
`test/helpers/llm-judge.ts:40-65` does not set temperature and extracts JSON from plain text via regex. Eval trend comparison (`bun run eval:compare`) implicitly assumes scores are reproducible. They are not. Two identical runs can produce different scores, and small variance could be misread as regression or improvement.

**Fix**: pass `temperature: 0` and use `tool_choice` with a JSON schema. Alternatively, run each judge call N times and use median.

### W3. Resolver files are too big
`scripts/resolvers/review.ts` (54K), `scripts/resolvers/design.ts` (58K), `scripts/resolvers/testing.ts` (27K). These aren't just long — they mix unrelated review concerns (SQL safety, race conditions, LLM output trust, enum completeness) into single files. Navigation is painful; targeted changes risk unintended side effects on adjacent sections.

**Fix**: split by concern. `review-sql.ts`, `review-concurrency.ts`, etc., each exported from `review/index.ts`. Preserves the `{{REVIEW_ARMY}}` interface.

### W4. No automated security test suite for the classifier ensemble
The 6-signal security stack is documented, the thresholds are tunable via env vars, but the evidence for "does a known attack actually get blocked?" is scarce. The team fixed the Stack Overflow FP with an ensemble rule — there should be a red-team fixture set proving it.

**Fix**: `test/security-ensemble.test.ts` with canary leak fixtures, L4 true positives, L4b true positives, and known Stack Overflow FP cases.

### W5. No per-PR spend limit
`EVALS_ALL=1` runs every eval. Diff-based selection helps the normal case. A developer who touches a "global touchfile" (`session-runner.ts`, `eval-store.ts`, `touchfiles.ts` itself) runs every test. Each run adds up. There's no ceiling, no confirmation, no kill switch. A malformed or accidentally-too-broad diff could burn a significant budget before anyone notices.

**Fix**: a configurable `GSTACK_MAX_PR_COST_USD` that aggregates across all test runs invoked in a single `bun test:evals` session and aborts if exceeded.

### W6. Chromium is a single-process singleton
One browser per user. `BROWSE_TAB` lets skills scope to tabs, but nothing prevents two concurrent Claude Code sessions from stepping on each other's tab state. Pages have cookies, localStorage, focus, scroll position — all shared. In a world where multiple skill windows are common, this is fragile.

**Fix**: per-session browser contexts. Playwright supports `browser.newContext()` with isolated storage state. Each Claude session could get its own context.

### W7. @refs expire silently
On navigation, server-side locators become stale. The next `click @e3` throws a Playwright error. The agent may or may not re-snapshot based on how the skill prompt is written. There's no automatic invalidation signal, no forced-snapshot wrapper, no structured "ref expired" response that the agent can reliably detect.

**Fix**: bind ref validity to a snapshot token that the server tracks. Return a typed `{error: "STALE_REF", newSnapshotRequired: true}` that skill templates can handle uniformly.

### W8. Skill templates are the single largest failure mode
Every new skill adds to the prompt surface. Each skill can drift from the team's voice, the preamble conventions, or the resolver contract. The `bun run skill:check` dashboard catches stale generation, but not semantic drift — no skill says "review a $500M M&A memo" but nothing in the pipeline would stop it from being written.

**Fix**: a voice-and-scope validator that reads every SKILL.md and runs an LLM-judge on "does this belong in a tool for founders shipping software?"

### W9. Confusion protocol has no termination condition
The preamble instructs Claude to AskUserQuestion when confused. There's no rule that says "after two consecutive asks, escalate or proceed with a best guess." A slow-responding user + an anxious model could produce an indefinite wait loop.

**Fix**: explicit "max three asks before proceeding with labeled assumption" clause in the protocol generator.

### W10. Binary-in-git debt
`browse/dist/` and `design/dist/` contain Mach-O arm64 binaries (~58MB each) that don't work on Linux/Windows/Intel. They're tracked by git due to a historical mistake and survive because `git add .` sweeps them in. CLAUDE.md flags this and instructs contributors to add by name — but that's convention, not enforcement.

**Fix**: pre-commit hook or CI check that fails if `dist/` is staged; plus a one-time `git rm --cached browse/dist/ design/dist/`.

---

## Hidden Coupling

### HC1. Skill templates know about $B
Every skill template that touches the browser embeds `$B` as a shell alias. If the alias is renamed, every skill is broken until regenerated. The preamble resolver injects the alias; the skill templates rely on the resolver. This is a one-way dependency but an invisible one.

### HC2. Resolver output depends on filesystem state
`generate-writing-style.ts` loads `scripts/jargon-list.json` at generation time. If the JSON file is deleted, the preamble silently drops the glossing list (try-catch with empty fallback, by design). This is a silent degradation path: skill behavior changes without a regeneration error.

### HC3. HostConfig.suppressedResolvers couples hosts to resolver names
Each host can suppress specific resolver outputs. This is done by string matching against resolver keys. If a resolver is renamed, a previously-suppressed resolver silently starts emitting on that host. No type-level protection.

### HC4. session-state.json is the only sidebar-agent / server communication channel
Both processes read and write the same JSON file. Neither uses file locks. Race conditions are unlikely (security state has a clear writer: the classifier subprocess writes, the server reads) but the contract is informal.

### HC5. `gstack-slug` shell script derives the eval namespace
`eval-store.ts` invokes `bin/gstack-slug` to get `SLUG` for the project directory. If that script returns a different slug across machines (e.g., git remote URL differs between laptop and CI), evals get split across namespaces silently.

---

## Likely Scale Bottlenecks

| Under load of... | Bottleneck | Why |
|------------------|-----------|-----|
| Many Claude Code sessions on one machine | One Chromium instance | Tab contention, no per-session context |
| Many skills in one PR run | LLM judge API rate limits | Single retry on 429; no backoff; runs serially |
| Large diffs | Context window exhaustion | 40K-token skills + large diff + tool history |
| Long eval histories | JSON file enumeration | eval-store reads project dir every `eval:list` |
| Many attack attempts | JSONL append lock contention | No sync flush; concurrent appends |
| Many skills per host config | `gen-skill-docs.ts` runtime | Runs synchronously, file-by-file |

---

## Ambiguity and Failure Bottlenecks

| Ambiguity | Consequence |
|-----------|-------------|
| "Is this finding Critical or Medium?" (review) | Model discretion; reviews can drift |
| "Is this decision Mechanical/Taste/User Challenge?" (autoplan) | Auto-decides what should be surfaced, or vice versa |
| "Is this test failure pre-existing?" | CLAUDE.md explicitly warns Claude not to make this claim without evidence — but enforcement is prompt-only |
| "Should I re-snapshot before clicking?" | Skill-template-specific; no universal rule |
| "Is this injection attempt real or FP?" | Requires ensemble agreement; single-layer high-confidence degrades to WARN |
| "Is this version bump major/minor/patch?" | Asked at `/ship` time; user answers |

The common thread: **the agent decides based on prompt guidance**, not on code rules. That's correct for most of these, but each one is a controlled burn of model discretion that can fail silently.

---

## Reliability Weaknesses

- **No checkpoint-before-long-work**: `/ship` can spend 30+ minutes running through 19 steps. A daemon crash at step 12 forces a full re-run; there's no resume mechanism.
- **No transactional semantics for multi-file writes**: `/ship` writes VERSION + CHANGELOG + commits; if any step fails between the three, state is inconsistent.
- **No rate limit on subagent fanout**: review-army spawns N specialist reviewers in parallel. If one hangs, the coordinator waits.
- **No retry policy on Supabase telemetry**: dropped events are dropped.

## Observability Weaknesses

- **No trace IDs**: a single `/ship` invocation is not correlated across logs, telemetry events, eval results.
- **No central error budget**: the attack log rotates at 10MB; evals rotate never; telemetry depends on Supabase retention.
- **stdout is the primary log sink**: if the daemon detaches and no one captured stdout, logs are lost.
- **No structured error categories**: errors surface as strings parsed by pattern matching.

## Context/Memory Weaknesses

- **`TodoWrite` is in-context**: lost if Claude restarts mid-workflow.
- **Learnings retrieval is file-based grep**: no semantic similarity; duplicate learnings accumulate.
- **No context compression policy for evicted history**: Claude's built-in compaction handles it, but gstack has no say in what gets kept.
- **Large skills crowd the context**: `/ship` at 42K tokens + large diffs on a 200K model = squeeze.

## Security / Control Weaknesses

- **Codex file boundary is prompt-only**: the instruction "don't read `~/.claude/`" is trust-based. A malicious prompt injection could override it.
- **No signed skill distribution**: `./setup` copies whatever's in the repo into `~/.claude/skills/`. A supply-chain compromise of the gstack repo reaches every user on their next `/gstack-upgrade`.
- **Security kill switch leaves canary running**: `GSTACK_SECURITY_OFF=1` turns off ML, but canary still runs. Good — but easy to misunderstand as "fully off."
- **No allowlist for which skills can spawn subagents**: any skill with `Agent` in its allowed-tools can fan out.

## Developer Workflow Weaknesses

- **Skill template changes are live immediately via symlink**: the dev symlink at `.claude/skills/gstack` means a broken `.tmpl` edit can break other concurrent Claude Code sessions.
- **Mechanical-vs-intentional changes bleed together**: the "bisect commits" norm is strong in CLAUDE.md, but nothing enforces it.
- **No sandboxed skill playground**: every test runs in a real worktree or a real subprocess; no in-memory preview for skill iteration.

---

## Where the Architecture Is Elegant

1. **The split**: prompts for decisions, code for mechanism, daemon for latency, extension for headed mode. Each piece does one thing.
2. **Single template → 10 hosts**: the host-aware compile step is cheap and correct.
3. **ref-based browser**: elides a whole class of agent errors without adding complexity.
4. **Tier-based preamble**: composition, not duplication.
5. **Physical port separation**: trivial in code, uncheatable in practice.

## Where the Architecture Is Fragile

1. **Module boundary invariants enforced only by comments**.
2. **Large resolver files that mix concerns**.
3. **Single Chromium singleton across concurrent sessions**.
4. **Stochastic LLM judge treated as comparable**.
5. **Prompt-only enforcement for "don't read ~/.claude"** trust boundaries.
6. **30-min idle shutdown + no checkpoint**: a long skill that gets through a long pause can lose its daemon mid-run.

---

## What I Would Harden First

If I had one week and had to reduce the highest-likelihood production failures:

1. **Add import-boundary tests**: a 20-line check that `server.ts` does not import `security-classifier.ts` and that `sse-session-cookie.ts` does not import `token-registry.ts`. Load-bearing invariants must be mechanically enforced.
2. **Set `temperature: 0` on LLM judge** and switch to structured output. Eval trends become meaningful.
3. **Per-PR eval budget kill switch**. Aggregate spend across `bun test:evals`, abort if over threshold.
4. **Red-team test suite for the security ensemble**. Fixtures: canary-leak, known injection payloads, Stack Overflow false positives. Ensemble has one job; verify it does that job.
5. **Resume-from-checkpoint in `/ship`**: each step writes a state record; on re-invocation, the skill resumes from the last completed step.
6. **Per-session Playwright browser contexts**: one Chromium instance, isolated contexts per Claude session. Kills the concurrent-skill contention class.
7. **Remove `dist/` from git**: one-time cleanup + pre-commit block.

Everything else can wait. These are the seven items that would most reduce "silent failure in production."
