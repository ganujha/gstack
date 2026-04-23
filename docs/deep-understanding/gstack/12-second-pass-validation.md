# 12-second-pass-validation.md — gstack

> Scope: Review every material first-pass claim. Mark each as **Confirmed from code**, **Confirmed from config**, **Confirmed from dependency manifest**, **Confirmed by execution**, **Strong inference**, **Weak inference**, or **Unverified**. Where the first pass was wrong or too strong, revise it and show the evidence.

Evidence labels follow the second-pass convention; citations reference lines from `/home/user/gstack/`. Runtime-dependent claims are all labeled `Unverified` because no `~/.gstack/` state directory exists in this sandbox.

---

## Claims Reviewed — Summary Table

| # | Claim (first-pass shorthand) | Verdict |
|---|------------------------------|---------|
| 1 | Browse daemon is a Bun.serve HTTP server | Confirmed from code |
| 2 | Dual-listener topology (local + tunnel) | Confirmed from code — **refined** |
| 3 | Allowlist is a "locked command allowlist" | Refined — it's two allowlists (path + command), both per-path enforcement |
| 4 | Six security layers (L1-L6) | Partial — revised to "three content-side layers + two ML classifiers in sidebar-agent + one deterministic canary + one verdict combiner" |
| 5 | L4b is a second separate Haiku file | Wrong — same file (`security-classifier.ts`), different method |
| 6 | `sidebar-agent.ts` is a separate subprocess | Confirmed from code |
| 7 | `security-classifier.ts` cannot be imported from the compiled binary | Confirmed from code (explicit file-top comment) |
| 8 | LLM judge uses `claude-sonnet-4-6` | Confirmed from code |
| 9 | LLM judge uses temperature 0 | **Wrong** — temperature is not set; API default (~1.0) |
| 10 | LLM judge uses structured JSON output | **Wrong** — regex extraction on plain text |
| 11 | Cost is ~$4/run gate, ~$3.85/run E2E | Partial — these are *estimates*; real cost is computed dynamically from token usage × model-specific prices |
| 12 | Cost caps per test not enforced | Confirmed from code (no budget kill-switch; only `max-turns` and `timeout`) |
| 13 | Preamble has 4 additive tiers | Confirmed from code |
| 14 | 22 preamble generators in `preamble/` | Confirmed from filesystem — **refined**: 22 files exist but one (`generate-test-failure-triage.ts`) is NOT composed into the preamble, it is exported as a standalone `{{TEST_FAILURE_TRIAGE}}` placeholder |
| 15 | Resolvers are a static `Record<string, ResolverFn>` | Confirmed from code |
| 16 | Placeholder expansion is fail-fast on missing | Confirmed from code |
| 17 | Placeholder expansion supports nesting | **Wrong** — no recursion; a resolver that emits `{{FOO}}` would leave literal text |
| 18 | 160KB token ceiling is advisory | Confirmed from code (warning only, no `exit(1)`) |
| 19 | Diff-based test selection via touchfiles | Confirmed from code |
| 20 | Global touchfiles trigger all tests | Confirmed from code (`GLOBAL_TOUCHFILES` array at touchfiles.ts:457) |
| 21 | E2E_TIERS classification for gate vs periodic | Confirmed from code |
| 22 | Session runner spawns `claude -p` NDJSON | Confirmed from code |
| 23 | Server startup uses atomic `wx` file lock | Confirmed from code |
| 24 | State file written atomically via `.tmp` rename | Confirmed from code |
| 25 | 30-minute idle shutdown exists | Confirmed from code (`IDLE_TIMEOUT_MS = 1_800_000`, checked every 60s) |
| 26 | Headed mode and tunnel mode skip idle shutdown | Confirmed from code |
| 27 | Windows uses a Node.js fallback server | Confirmed from code + shell script |
| 28 | Supabase powers telemetry backend | Confirmed from code (it does more — see #44) |
| 29 | Supabase uses service role key | **Wrong** — uses anon key + RLS (deliberate, commented at telemetry-ingest/index.ts:47-52) |
| 30 | Skill content is ~40K tokens max | Confirmed from config (warning threshold) |
| 31 | `/ship` skill has 19 steps | Confirmed from skill template |
| 32 | `/autoplan` is sequential CEO→Design→Eng→DX | Confirmed from skill template |
| 33 | `/investigate` has a 3-strike rule | Confirmed from skill template |
| 34 | Sidebar model routing is a regex heuristic (Opus for analysis, Sonnet for actions) | Confirmed from code (server.ts:193-211) |
| 35 | Canary is deterministic + always overrides ensemble | Confirmed from code |
| 36 | `combineVerdict` requires 2-of-N ML classifiers at WARN | Confirmed from code |
| 37 | Tunnel allowlist is "a subset of command surface" | Refined — it is *two* allowlists enforced at different layers (path at route dispatch, command at POST body) |
| 38 | `@ref` resolution is server-side locator map | Confirmed from code |
| 39 | `@ref` is invalidated on navigation | Unverified — Playwright page object changes would fail the locator silently; no explicit invalidation event found in snapshot.ts skim |
| 40 | No retry on `claude -p` subprocess failure | Confirmed from code |
| 41 | `maxTurns` default is 15 | Confirmed from code |
| 42 | Eval store falls back to `~/.gstack-dev/evals/` when slug detection fails | Confirmed from code |
| 43 | Cost estimation uses model-specific pricing × token usage | Confirmed from code (`test/helpers/pricing.ts`) |
| 44 | Supabase is "analytics/telemetry, inferred" | Refined — **three** edge functions: `telemetry-ingest`, `community-pulse`, `update-check`. Not just telemetry: also an update-check ping and community metrics |
| 45 | gbrain is an external AI service | Still inferred — `hosts/gbrain.ts` and `scripts/resolvers/gbrain.ts` exist but external endpoint is not visible in the repo |
| 46 | Conductor is an external product | Unverified — `conductor.json` is a config file consumed by the Conductor product (multi-workspace git tool) but gstack does not ship Conductor code |
| 47 | Learnings are JSONL | Unverified — no runtime `~/.gstack/learnings/` to inspect |
| 48 | Security state is cross-process via JSON file | Confirmed from code (referenced by both server.ts and sidebar-agent.ts) |
| 49 | Skill validation catches stale placeholders | Confirmed from code |
| 50 | `bun run skill:check` runs dry-run regen + diff | Confirmed from code |

---

## Revised Claims (with evidence)

### R1. Security layer count (was: "6 layers")

- **Original claim**: 6 distinct security layers (L1 datamarking, L2 hidden strip, L3 URL filter, L4 TestSavantAI, L4b Haiku transcript, L5 canary, L6 combineVerdict).
- **Revised claim**: Three content-side transforms + one content-ML classifier + one transcript-ML classifier + one deterministic canary + one verdict combiner. The "6" is a reasonable operator mnemonic but flattens two different **categories** of defense: (a) pre-inference transforms of untrusted content, (b) ML scoring, (c) deterministic tripwire, (d) decision fusion. L6 is not a defensive layer — it is the integration point that consumes L4/L4b/L5 signals.
- **Why the revision matters**: For critique purposes, L1-L3 always run (deterministic, fast, in-process); L4/L4b run only inside `sidebar-agent.ts` (separate process, model-dependent, ~10-50ms + Haiku latency); L5 is always deterministic. Confusing "layers" with "checks" makes the cost/latency profile hard to reason about.
- **Evidence**: `browse/src/security.ts:9-14` comment block; `browse/src/security-classifier.ts:2-10` (same module exports both `classifyContent` and `checkTranscript`); `browse/src/sidebar-agent.ts:16-27, 496` (both invoked from the subprocess).

### R2. L4b is NOT a separate file

- **Original claim**: "L4b Claude Haiku transcript classifier" implied a second file.
- **Revised claim**: `security-classifier.ts` exports two functions (`classifyContent` via TestSavantAI ONNX; `checkTranscript` via Anthropic Haiku API). Both run only inside `sidebar-agent.ts`. Naming them L4 and L4b is fine — just don't assume separate modules.
- **Evidence**: `browse/src/security-classifier.ts:2-10`; `browse/src/sidebar-agent.ts:23-27, 496`.

### R3. LLM judge temperature and structure

- **Original claim**: Judge uses temperature 0 (assumed); structured output (assumed).
- **Revised claim**: Judge does **not** set temperature — uses API default. It does **not** use structured output or tool-choice JSON schemas; it extracts JSON from a plain-text completion via regex `text.match(/\{[\s\S]*\}/)` then `JSON.parse`. Malformed responses could silently fail or throw.
- **Why the revision matters**: Reproducibility of LLM-judge scores is weaker than assumed. A non-zero temperature + regex extraction means judge scores can drift across identical inputs. This is relevant to any claim that eval scores are comparable across runs.
- **Evidence**: `test/helpers/llm-judge.ts:40-65` (`callJudge`).

### R4. Cost model is dynamic, not fixed

- **Original claim**: Gate evals cost ~$4/run; E2E ~$3.85/run.
- **Revised claim**: Cost is computed per-run as `(input × price) + (cached × price × 0.1) + (output × price)` using `test/helpers/pricing.ts`. The "~$4" figure is a *headroom budget* mentioned in CLAUDE.md, not an accounting constant. Claude CLI's own `total_cost_usd` in `resultLine` is used when available. This means eval costs scale with diff size and turn count, not with test count.
- **Evidence**: `test/helpers/pricing.ts:20-61`; `test/helpers/session-runner.ts:~350` (resultLine.usage consumption).

### R5. Placeholder expansion is non-recursive

- **Original claim**: `resolveTemplate` expands placeholders (no statement about recursion).
- **Revised claim**: The regex `/\{\{(\w+(?::[^}]+)?)\}\}/g` is matched once against the whole file. Resolvers emit final text — if a resolver returns a string containing `{{FOO}}`, the token will appear literally in the output. A post-pass check (line 445-447) throws if any `{{...}}` remains, so this bug would be caught at build time — but only if the placeholder matches the regex. A subtly corrupted output (e.g., `{{FOO:bar}}{{` truncation) might slip.
- **Why it matters**: The resolver library is a compile step with one level of composition, not a general macro system. Adding nested macros would require wrapping the regex loop in a `while (changed) { ... }` convergence pass, with a depth cap.
- **Evidence**: `scripts/gen-skill-docs.ts:430-448`.

### R6. Tunnel allowlist is path + command, not just command

- **Original claim (CLAUDE.md-derived)**: "tunnel listener serves a locked 17-command allowlist."
- **Revised claim**: The tunnel surface enforces TWO allowlists:
  1. **Path allowlist** at route dispatch (`/connect`, `/command`, `/sidebar-chat`) — other paths return 404 before any handler runs. This is server.ts:95-99 + server.ts:1568-1590.
  2. **Command allowlist** inside the `/command` handler (17 browser-driving verbs). This is server.ts:108-113 + server.ts:2533-2541.
- **Why the revision matters**: Defense in depth. Even if a handler for `/command` were compromised, the `/command` path wouldn't leak access to `/admin` or `/cookies` endpoints that the local surface exposes.
- **Evidence**: `browse/src/server.ts:95-113, 1568-1590, 2533-2541`.

### R7. Supabase scope is wider than "telemetry"

- **Original claim**: Supabase is likely analytics/telemetry.
- **Revised claim**: Three edge functions: `telemetry-ingest` (batch event insert), `community-pulse` (aggregate read; for "community metrics" surfacing in skills), `update-check` (version ping). Telemetry writes use the **anon key + RLS policies**, not the service_role key — a good security decision with a deliberate comment at `supabase/functions/telemetry-ingest/index.ts:47-52`.
- **Why the revision matters**: gstack has a real server-side surface with three behaviors, not just a fire-and-forget telemetry drain. The community-pulse endpoint could become a cross-user influence channel worth reviewing for privacy.
- **Evidence**: `ls supabase/functions/`; `supabase/functions/telemetry-ingest/index.ts:1-60`.

### R8. Sidebar model routing is a regex, not an ML decision

- **Original claim**: "pickSidebarModel regex heuristic".
- **Revised claim (confirmed + quantified)**: Three regexes — `ANALYSIS_WORDS` (routes to Opus), `ACTION_PATTERNS` (short imperative starting with verb → Sonnet), `ACTION_ANYWHERE` (action verb anywhere → Sonnet). Default: Opus. Comment in code states the motive: "sonnet vs opus on 'click @e24' is 5-10x in latency and cost, with zero quality difference."
- **Why it matters**: This is a cost-tier routing pattern that can be reused in other agent systems. It deserves a best-pattern callout.
- **Evidence**: `browse/src/server.ts:190-211`.

### R9. State writes are atomic; in-flight risk is different

- **Original claim**: "State file writes may not be atomic".
- **Revised claim**: `browse.json` IS written atomically via `.tmp → rename` (server.ts:2728-2730). Risk of corruption from partial write is therefore zero. The *actual* concurrency risk is: (a) two processes reading before the second writes and overwriting a newer PID; (b) the lock is held only during *startup*, not during steady-state state updates. Two Claude Code sessions invoking `$B` simultaneously could both read a stale state, both ping /health on an obsolete port, and race to start. The `wx` lock prevents that for the "no server yet" case but not for "version mismatch requiring restart" paths.
- **Evidence**: `browse/src/server.ts:2728-2730`; `browse/src/cli.ts:221-306` (lock scope).

### R10. `security.ts` has a pure-string module boundary

- **Original claim (implicit)**: security primitives are spread across multiple files.
- **Revised claim (promoted to architectural note)**: `security.ts` is deliberately pure-string-ops (canary, verdict combiner, attack log, status) so it can be imported from the compiled browse binary. All ML/async classification lives in `security-classifier.ts`, which is only imported from the non-compiled `sidebar-agent.ts` subprocess. The architectural invariant — "no imports from `token-registry.ts` into `sse-session-cookie.ts`, and no imports of `security-classifier.ts` into `server.ts`" — is load-bearing. A second-pass observer would flag: this invariant is enforced only by convention + a CLAUDE.md callout, not by eslint or a dependency-cruiser rule.
- **Evidence**: `browse/src/security.ts:1-20` and `browse/src/security-classifier.ts:2-10`; no lint rule in `package.json`.

---

## New Evidence That Changed the Interpretation

1. **Temperature on LLM judge**: Makes eval reproducibility weaker than assumed. Follow-up: check variance across identical re-runs.
2. **Static `Record<string, ResolverFn>` with fail-fast**: Makes the skill pipeline simpler than a "plugin system" framing; it's essentially a typed string-interpolation compiler with a validation post-pass.
3. **Dynamic cost accounting**: The diff-based selection system is even more efficient than a "skip test" model — it skips nothing if a touchfile changed, but pays linearly in what it runs.
4. **Sidebar model routing regex**: A tiny (~15-line) function is doing the work of what many agent frameworks use an ML classifier for. This is a powerful simplicity argument.
5. **Module-boundary conventions for ONNX**: The compiled-binary limitation pushed the team toward a multi-process architecture where the ML-heavy work is isolated. This is a very consequential architectural choice driven by a tool limitation.
6. **Supabase scope**: Three functions, not one. The anon+RLS pattern is a security best practice worth surfacing.

---

## Remaining Unknowns

| # | Topic | Why unresolved |
|---|-------|----------------|
| U1 | `@ref` invalidation on navigation — explicit event handler or lazy-fail? | Not found in first-pass code skim; requires reading `browser-manager.ts` in detail |
| U2 | Learnings directory structure | No runtime `~/.gstack/learnings/` present; only derived from resolver code |
| U3 | Concurrency behavior across multiple Conductor workspaces | Claim is "10 workspaces, zero conflicts" via random ports; not tested in this sandbox |
| U4 | ML classifier cold-start behavior on first-run model download | Requires running with empty `~/.gstack/models/` — no sandbox to verify |
| U5 | Whether `combineVerdict` emits any structured signal the caller can override mid-request | Need to read `security.ts:96-178` in full |
| U6 | CI-time dedup policy for git worktrees | `lib/worktree.ts` has a dedup index; retention after a failed run is unclear |
| U7 | Exact behavior of version-mismatch restart when two CLI instances detect mismatch simultaneously | Possible double-kill race |
| U8 | Real-world false-positive rate of ML classifiers on code-editing pages | Not testable without dataset + runtime |
| U9 | Whether 40+ skills are actually all maintained or some are abandoned | First-pass pattern counts don't imply active use |
| U10 | Whether `gbrain` refers to an external agent runtime or is a host-config placeholder for future work | `hosts/gbrain.ts` contents not fully read |

---

## What To Carry Forward Into the Remaining Second-Pass Docs

1. The security stack is actually **two-tiered** (fast always-on + slow subprocess), not a flat 6-layer stack. The critique and production-readiness docs should use this framing.
2. The LLM judge is stochastic. The tradeoff doc should surface "reproducibility vs. cost of deterministic rescoring."
3. The cost model is dynamic. The production-readiness doc should recommend a budget kill-switch rather than static cost caps.
4. The resolver system is a single-pass typed compiler. It's deliberately NOT a macro system. Interview questions should probe why, and candidates should not try to sell "support nested macros" as a safe improvement.
5. Two module boundary invariants (`security-classifier.ts ⊄ server.ts`; `token-registry.ts ⊄ sse-session-cookie.ts`) are load-bearing but only doc-enforced. This is a real reliability risk.
