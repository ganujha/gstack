# 18-interview-drills.md — gstack

> 40 questions grounded in gstack's actual architecture. Each question includes why it's relevant to this repo specifically, what a strong answer must cover, and common weak-answer patterns to avoid.

---

## System Design Questions (10)

### SD1. Design a persistent daemon that fronts a headless browser for an AI agent.
**Relevance**: `browse/src/server.ts` + `cli.ts` is this problem. Interviewer can compare candidate answers to gstack's actual design.
**Strong answer must cover**:
- Cold-start cost justifies a persistent process (2-3s → <100ms per command).
- State file (`pid`, `port`, `token`) with atomic write semantics.
- Client health check + retry + lock for concurrent startup.
- Idle shutdown to reclaim memory.
- Auth token on every call.
- How to expose a subset of the surface to remote clients (port separation, not headers).
**Weak pattern**: "I'd use a singleton in Node.js." Missing process model; missing auth; missing startup lock.

### SD2. Design a multi-host prompt distribution system for 10 AI agents from one template.
**Relevance**: `scripts/gen-skill-docs.ts` + `hosts/*.ts`.
**Strong answer must cover**:
- Typed host config with path rewrites, tool rewrites, frontmatter transforms.
- One template → N outputs pipeline; fail-fast on missing resolvers.
- Per-host frontmatter allowlist/denylist.
- CI freshness check.
- Token budget warning.
**Weak pattern**: "I'd parse the template in the agent at runtime." Misses the compile step entirely.

### SD3. Design prompt injection defense for an AI agent that reads untrusted web content.
**Relevance**: `browse/src/content-security.ts` + `security.ts` + `security-classifier.ts`.
**Strong answer must cover**:
- Multiple signal types (content transforms + ML + canary).
- Ensemble rule to avoid single-classifier FP.
- Deterministic canary as tripwire.
- Fast path (always-on, pure-string) separated from expensive path (ML in a subprocess).
- Kill switch with audit trail.
**Weak pattern**: "I'd run an LLM to check for injection." Missing ensemble, missing determinism, missing cost/latency separation.

### SD4. Design diff-based test selection for expensive evaluation suites.
**Relevance**: `test/helpers/touchfiles.ts`.
**Strong answer must cover**:
- Per-test dependency declaration.
- Global touchfiles that trigger everything.
- Env-var override (`EVALS_ALL=1`).
- Tier classification (gate vs periodic).
- Preview command (`eval:select`) for debugging.
**Weak pattern**: "I'd use `git diff` and heuristics." Too vague; misses the explicit dependency declaration.

### SD5. Design a skill/prompt compile pipeline with type safety at build time.
**Relevance**: the resolver system.
**Strong answer must cover**:
- Placeholder regex (`{{KEY}}` or `{{KEY:arg}}`).
- Resolvers as typed functions returning strings.
- Registry as `Record<string, ResolverFn>`.
- Fail-fast on missing placeholders after expansion.
- No recursion by default (explicit choice).
- Side-channel resources (e.g., `jargon-list.json`) loaded at compile time, not runtime.
**Weak pattern**: "Like handlebars." OK if they know handlebars well; weak if they don't articulate the typing and fail-fast behavior.

### SD6. Design dual-listener topology for a local HTTP service that also needs remote tunnel access.
**Relevance**: `browse/src/server.ts` two `Bun.serve` calls.
**Strong answer must cover**:
- Physical port separation (not header-based origin).
- Path allowlist enforced before handler dispatch.
- Command allowlist enforced inside allowlisted path.
- Tunnel-scoped auth tokens separate from local tokens.
- Denial logging.
**Weak pattern**: "Check Origin header." Spoofable; insufficient defense.

### SD7. Design cost accounting for per-call LLM-based evaluations.
**Relevance**: `test/helpers/pricing.ts` + `eval-store.ts`.
**Strong answer must cover**:
- Per-model pricing table (input, output, cached).
- Use `resultLine.usage` from the CLI or API response.
- Persist per-run cost for trend analysis.
- Aggregation for dashboards.
- Budget kill switch (gstack does not have this; strong candidate should propose it).
**Weak pattern**: "Multiply tokens by $0.003." Missing cached-token discount, missing per-model differences.

### SD8. Design a worktree isolation system for parallel tests that must not see each other's changes.
**Relevance**: `lib/worktree.ts`.
**Strong answer must cover**:
- `git worktree add` for per-test isolation.
- Dedup by patch hash (SHA256).
- Harvest (stage + diff) after run.
- Cleanup policy with stale-worktree pruning.
**Weak pattern**: "Each test makes a copy of the repo." Expensive; doesn't use git's native isolation.

### SD9. Design an in-context-scratchpad + persistent-filesystem + cross-session-learnings memory hierarchy.
**Relevance**: `TodoWrite`, `~/.gstack/projects/*/evals/`, `~/.gstack/learnings/`.
**Strong answer must cover**:
- Tier boundaries (ephemeral session, per-project persistent, cross-project persistent).
- Access patterns (read-on-entry, write-on-decision, grep-on-search).
- Why no vector DB (debuggability, privacy, infrastructure).
- When embeddings become justified (at N projects × M learnings).
**Weak pattern**: "Chroma/pgvector." Jumping to embeddings without justifying why files-and-grep is insufficient.

### SD10. Design a "chat sidebar" that routes between cheap and expensive models based on user intent.
**Relevance**: `browse/src/server.ts:pickSidebarModel`.
**Strong answer must cover**:
- Intent classification cheap enough not to need its own ML.
- Regex pattern for imperative actions → cheap model.
- Default to powerful model on ambiguity.
- Observability on routing decisions.
**Weak pattern**: "Train a BERT intent classifier." Over-engineered; gstack gets the same result with 15 lines of regex.

---

## Debugging / Reliability Questions (10)

### DR1. A user reports that `$B click @e3` fails intermittently after navigation. What do you investigate?
**Relevance**: @ref invalidation semantics.
**Strong answer**: Server-side locator map is stale; Playwright throws on the old locator. Check whether the skill template includes a re-snapshot before write commands. Propose: a structured "stale ref" error type returned to the agent.
**Weak**: "Add retries."

### DR2. The browse daemon exits after 30 minutes of use during a long skill. How do you diagnose and prevent?
**Relevance**: idle shutdown timer.
**Strong answer**: `IDLE_TIMEOUT_MS`; `resetIdleTimer` on each `/command`. Long workflows that pause for user input don't reset. Fix: add "in-progress skill" signal from the CLI that pauses the timer.
**Weak**: "Increase the timeout."

### DR3. Evals pass on one run and fail on re-run with identical code. Likely causes?
**Relevance**: LLM judge is stochastic.
**Strong answer**: `llm-judge.ts` doesn't set temperature; uses regex JSON extraction. Propose `temperature: 0` + structured output. Also: model version rolls, non-deterministic `claude -p` turn sequences.
**Weak**: "Flaky tests."

### DR4. Two Claude Code sessions on the same machine seem to interfere with each other during browser commands. What's happening?
**Relevance**: singleton Chromium + shared tab state.
**Strong answer**: One Chromium process per user; `BROWSE_TAB` env var for scoping is opt-in. Tab focus/scroll/cookies leak. Propose per-session Playwright `browser.newContext()`.
**Weak**: "Add mutexes."

### DR5. ML classifier blocks a legitimate Stack Overflow page. What's the likely cause, what's the correct fix?
**Relevance**: documented Stack Overflow FP case.
**Strong answer**: Single classifier fires high-confidence on instruction-writing prose. The `combineVerdict` ensemble rule degrades single-high-confidence to WARN. Check `WARN = 0.60` threshold. Verify canary wasn't leaked. Fix: ensure ensemble is active; consider adding known-good fixture to prevent regression.
**Weak**: "Disable the classifier."

### DR6. After `./setup`, `$B` commands return "connection refused" and the daemon won't start.
**Relevance**: startup lock path; state file; Playwright missing.
**Strong answer**: Check `~/.gstack/browse.json.lock` — stale lock from crashed process. Check for Playwright chromium install. Inspect daemon stdout (captured if attached; lost if detached). Propose: structured log file, not stdout.
**Weak**: "Reinstall."

### DR7. A `/ship` invocation loses progress mid-way. What's missing from the architecture?
**Relevance**: no skill-level checkpoint/resume.
**Strong answer**: The 19-step ship has no disk-backed progress record; Claude loses context on restart. Fix: each step writes a checkpoint to `.gstack/ship-state.json`; re-invocation resumes from last completed step.
**Weak**: "Re-run the skill."

### DR8. Concurrent `$B` calls from two shells produce inconsistent results. Root cause?
**Relevance**: Playwright serializes page ops; command ordering is non-deterministic; `BROWSE_TAB` scoping is optional.
**Strong answer**: Single browser singleton; implicit ordering races; tab focus can flip under you. Fix: scope each CLI invocation to a specific tab, or move to per-context isolation.
**Weak**: "Add a mutex in the handler."

### DR9. Eval cost triples suddenly with no code change. Where to look?
**Relevance**: no budget cap; model version roll; `max-turns` increase; failing tests hit turn ceiling.
**Strong answer**: `pricing.ts` compute with actual `resultLine.usage`. Inspect per-test deltas via `bun run eval:compare`. Check for model change in `session-runner.ts` (did the default model roll to Opus?). Check max-turns changes in tests. Propose budget kill switch.
**Weak**: "Cache more."

### DR10. Security classifier download fails on first run. User sees inconsistent security behavior. Diagnose and fix.
**Relevance**: no offline fallback documented; `GSTACK_SECURITY_OFF` kill switch exists.
**Strong answer**: 112MB model from HuggingFace; network failure → classifier cannot initialize. sidebar-agent should degrade gracefully (canary + content transforms still run); add retry with backoff; surface failure in `/health`. Propose: bundle a small fallback model in-repo or cache via CDN.
**Weak**: "Set `GSTACK_SECURITY_OFF=1`."

---

## Architecture Trade-off Questions (10)

### AT1. Why does gstack compile skills at build time instead of at runtime?
**Strong answer**: Type safety, fail-fast validation, consistent output across hosts, CI can check freshness, one template → many hosts. Cost: rebuild required; no runtime flexibility. Worth it because gstack ships updates via `./setup`, not live config.

### AT2. Why two processes for security (server + sidebar-agent) instead of one?
**Strong answer**: Forced by tool limitation (`@huggingface/transformers` can't `dlopen` ONNX from compiled Bun binary). Good side effects: ML cost is isolated; fast path stays fast; compile surface stays small. Bad side effects: cross-process state (session-state.json); architectural invariant enforced only by documentation.

### AT3. Why files-and-grep for learnings instead of a vector DB?
**Strong answer**: Debuggability, zero-infrastructure, privacy, inspectability. At current N (single user, tens of learnings per project), embeddings would be over-engineered. At N=100+ per project with cross-project search demand, a local SQLite-FTS or embeddings layer becomes justified.

### AT4. Why opinionated 19-step `/ship` instead of a minimal merge-commit script?
**Strong answer**: "Boil the Lake" philosophy — completeness is cheap with AI. Every ship gets CHANGELOG, version bump, canary check. Prevents "ship rushed" class of bugs. Cost: slow iteration loop. Mitigation: cheap if run in parallel with other work.

### AT5. Why dual-listener ports instead of an internal auth-header check on one port?
**Strong answer**: Headers can be spoofed or overridden by upstream proxies. Physical port separation means a tunnel caller literally cannot reach the local-only path set. Higher assurance for the cost of one extra socket bind.

### AT6. Why no agent framework (LangChain, AutoGen, etc.)?
**Strong answer**: Claude Code is the framework. gstack is the configuration + tool library. Adding a framework would duplicate Claude Code's agent loop. Cost: reinventing session-runner, touchfiles, eval-store. Benefit: no upstream lock-in, no framework-flavored abstractions to learn.

### AT7. Why per-PR diff-based test selection instead of running all evals?
**Strong answer**: $4 × full suite × per-PR × N PRs = real money. Touchfiles declare "which tests depend on which source." Changes trigger only affected tests. Global touchfiles (infra changes) trigger all. Trade: a bug in a test's touchfile declaration hides it.

### AT8. Why a regex for sidebar model routing instead of an ML classifier?
**Strong answer**: Regex is deterministic, free, sub-millisecond. For the well-defined intent split (action vs analysis), it's as accurate as a classifier. Start simple; upgrade when measurable signal exists that regex is missing cases.

### AT9. Why no mutual TLS / signed requests between CLI and daemon?
**Strong answer**: Both are on 127.0.0.1; same OS user. Bearer token already in state file (0o600). mTLS would add complexity without closing a real attack vector. Worth it only if the daemon ever serves over a non-loopback interface (ngrok is a separate, scoped surface).

### AT10. Why does the security layer have a kill switch env var?
**Strong answer**: Real-world escape hatch for false positives; research ensemble toggle. Cost: a compromised env disables defense silently; no audit. Acceptable for single-user installs. For team rollouts, the kill switch should be policy-gated.

---

## "Improve This System" Prompts (5)

### IM1. Add an import-boundary test.
**Goal**: Enforce "`security-classifier.ts` is not imported from `server.ts` or anything that reaches `server.ts`."
**Solution**: simple `bun test` that reads every source file's import list; fails if banned edges exist. Or use `dependency-cruiser` with a YAML ruleset.
**What a weak candidate might propose**: "Just add a comment." (Already has one; doesn't work.)

### IM2. Add a per-PR eval budget kill switch.
**Goal**: stop runaway cost.
**Solution**: env var `GSTACK_MAX_PR_COST_USD=10`; session-runner aggregates total cost across a `bun test:evals` invocation and aborts further runs if the threshold is crossed.

### IM3. Add `temperature: 0` + structured output to the LLM judge.
**Goal**: reproducible scores.
**Solution**: pass `temperature: 0` to Anthropic SDK; use tool-choice with a JSON schema.

### IM4. Add checkpoint/resume to `/ship`.
**Goal**: survive daemon crash or user interruption mid-workflow.
**Solution**: each ship step writes `{step: N, state: ...}` to `.gstack/ship-state.json`; on re-invocation, skill reads the file and resumes.

### IM5. Add per-Claude-session Playwright browser contexts.
**Goal**: prevent cross-session state contention.
**Solution**: daemon tracks `sessionId` per /command (derived from the CLI process environment); creates `browser.newContext()` on first use per session; destroys on idle.

---

## "What Would Break Under Scale?" Prompts (5)

### BR1. 10,000 skills instead of 40.
- `gen-skill-docs.ts` runs serially; build time balloons.
- Token ceiling warning fires on many skills.
- `bun run skill:check` dashboard is unmanageable.
- Fix: parallel generation, cumulative token budget UI, tiered dashboards.

### BR2. 100 concurrent Claude Code sessions on a single shared server.
- Singleton Chromium bottlenecks.
- State file contention.
- Port exhaustion on random-port allocation.
- Fix: per-session daemons or per-session Playwright contexts; port pool instead of random.

### BR3. 10,000 eval runs per day across a team.
- Eval store is a flat directory; `eval:list` enumerates all.
- Supabase ingest batch limit (100) requires batching.
- No central dashboard.
- Fix: retention policy; paged queries; aggregate dashboards.

### BR4. Security classifier used on 1M page loads per day.
- ONNX inference CPU cost dominates.
- Attack log grows fast.
- Haiku transcript classifier API cost becomes material.
- Fix: sample-and-cache; attack log rotation tuned tighter; selective classifier skipping for known-safe domains.

### BR5. Skill-template content is user-contributed.
- Resolver sprawl becomes unmaintainable.
- Voice drift across contributors.
- Security review surface scales with skill count.
- Fix: contributor skill contract (lint), voice-check LLM judge, skill review process.

---

## "How Would You Productionize This?" Prompts (5)

### PR1. Productionize the browse daemon for multi-user shared hosts.
Steps: per-user state dirs; per-user auth tokens; capability-based access; resource quotas; monitoring and alerting.

### PR2. Productionize the LLM judge for eval trend SLAs.
Steps: `temperature: 0`; structured output; run N times with median; persist variance; dashboard alerts on drift > X%.

### PR3. Productionize the multi-host skill distribution for a team that uses Codex primarily.
Steps: add Codex E2E runner; enforce runtime parity or deprecate Codex hosts; signed skill releases with verification in host's installer; per-host telemetry.

### PR4. Productionize `/ship` for a team landing 20 PRs a week.
Steps: checkpoint/resume; parallel canary verification; land-and-deploy subagent; conflict detection against simultaneous `/ship` runs.

### PR5. Productionize the security stack for enterprise pilot.
Steps: red-team test fixtures; managed model cache; SIEM integration for attack log; policy-gated kill switch; customer-specific classifier thresholds.

---

## Question Quality Notes

Every question above is specific to gstack's actual design choices — asking them about a random "AI agent tool" candidate would produce weaker answers because the candidate wouldn't have the concrete architecture to ground their reasoning. Use these as stress tests when you want to see how a candidate handles complexity grounded in a real system, not the abstract "design X" form.
