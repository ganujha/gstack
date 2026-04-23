# 15-tradeoff-analysis.md — gstack

> Every trade-off below names the choice gstack appears to make, the evidence, the benefits the choice buys, the downsides it carries, and the conditions under which a different choice would be justified.

---

## T1. Simplicity vs Extensibility

**The choice**: single-pass, typed, static resolver registry. No plugin system. No dynamic discovery. No hooks or middleware.

**Evidence**: `scripts/resolvers/index.ts:26` is a literal `Record<string, ResolverFn>`; placeholders expand once (`scripts/gen-skill-docs.ts:430-448`) with fail-fast on missing.

**Benefits**:
- Every skill compile is deterministic and debuggable. Output depends only on inputs.
- Type system catches typos at compile time. No runtime surprises from a missing handler.
- New developers can read the whole generator in under two hours.

**Downsides**:
- No third-party resolvers. If a YC founder wants a bespoke `{{FOUNDER_INTRO}}` section, they fork gstack.
- No composition across resolvers. A resolver that wants another resolver's output must duplicate it.
- Nested placeholders are literally unsupported.

**When a different choice would be justified**: at ~100 skills with community contributions, dynamic discovery becomes worth the complexity. At that scale, a plugin-per-directory pattern (each `/resolvers/<name>/` exporting a known symbol) would let independent teams ship without merging into core.

---

## T2. Speed vs Safety (Security Layer)

**The choice**: fast deterministic transforms (L1-L3) run everywhere; expensive ML classifiers (L4, L4b) run only in the sidebar-agent subprocess.

**Evidence**: `browse/src/security.ts` uses only pure-string operations; `browse/src/security-classifier.ts:2-10` is restricted to sidebar-agent.

**Benefits**:
- `$B <cmd>` latency stays sub-100ms even with security enabled.
- Heavy ML inference happens only on conversation surfaces where cost and latency are amortized against model inference anyway.
- Browse binary size stays reasonable; ONNX runtime stays in the process that tolerates its startup cost.

**Downsides**:
- Non-sidebar paths (plain `$B text`) get less defense. A skill that reads page content via `$B` and feeds it into a prompt is relying on L1-L3 only.
- Two different trust models (sidebar vs CLI) must be internalized by contributors.
- Architectural invariant enforced only by documentation.

**When a different choice would be justified**: if a new attack class emerges that L1-L3 can't catch, the team will need to either move classifiers into-process (requires solving the Bun compile + ONNX `dlopen` issue) or push sidebar-agent to be the only untrusted-content path.

---

## T3. Model Discretion vs Deterministic Enforcement

**The choice**: determinism for auth/transport/state; model discretion for review severity, version-bump type, decision classification.

**Evidence**: Auth checks, allowlists, canary checks, state writes are code-enforced. `/autoplan`'s Mechanical/Taste/User Challenge taxonomy, `/review`'s severity tiers, `/ship`'s version bump decision are prompt-driven.

**Benefits**:
- Model does what models are good at (judgment under ambiguity); code does what code is good at (verification, invariants).
- New review categories are prompt edits, not code changes.

**Downsides**:
- Inconsistent severity labeling across runs. No calibration.
- The confusion protocol has no code-level termination. An indefinitely confused agent is a prompt-only bug.
- "Is this test failure pre-existing?" is a prompt-level guardrail. CLAUDE.md warns against false claims here; enforcement is aspirational.

**When a different choice would be justified**: if review-severity scores were ever used for SLA triage (which they are not today), the team would need calibration rubrics + post-hoc scoring. Today the output is a human-reviewable report, not an actionable metric.

---

## T4. Local State vs Persistent Memory

**The choice**: filesystem + grep for memory; no vector database; Supabase only for telemetry and update checks.

**Evidence**: `~/.gstack/` is all local files. `bin/gstack-learnings-log` writes locally. `supabase/functions/` has telemetry-ingest, community-pulse, update-check — no learning store.

**Benefits**:
- Memory is debuggable: `cat` / `grep` / `jq`.
- No infrastructure dependency; the tool works offline.
- Privacy: the user's learnings never leave their machine.
- No embedding cost, no vector DB TCO.

**Downsides**:
- No semantic retrieval. "Did we hit this bug before?" depends on exact keyword matches.
- Cross-project pattern sharing is manual. You can't search across your installation base for "other projects that hit this Bun issue."
- Scale ceiling: after a year of skill use, learnings/ directory is a flat file pile with no structure.

**When a different choice would be justified**: when the learnings-search signal degrades noticeably — i.e., a user reports "I know I solved this two months ago but I can't find it." At that point, add a local SQLite-FTS index (not a vector DB — still local, still inspectable) before reaching for embeddings.

---

## T5. Single-Agent vs Multi-Agent Complexity

**The choice**: primarily single-agent with optional sequential multi-phase (autoplan) and parallel specialist fanout (review).

**Evidence**: `scripts/resolvers/review-army.ts` generates a prompt that instructs Claude to spawn parallel reviewers via the Agent tool. `autoplan/SKILL.md.tmpl` runs CEO → Design → Eng → DX serially.

**Benefits**:
- No orchestration framework to maintain.
- State lives in the file system and in the current Claude context; simple mental model.
- Parallel fanout is cheap to spin up (prompt + Agent tool call).

**Downsides**:
- No inter-subagent communication: each reviewer is isolated; they can't disagree with each other in real time.
- No timeout on subagents; a hung reviewer blocks the coordinator.
- No result aggregation policy; the coordinator must be told how to merge reviewer outputs.
- Every subagent pays full context cost (no shared prompt caching across subagents — Claude Code handles caching per-session).

**When a different choice would be justified**: if gstack ever needs agents that negotiate (e.g., planner proposes, executor rejects, they iterate), an explicit multi-agent graph (LangGraph-style) would be necessary. Today, the review fanout is read-only aggregation, which doesn't need negotiation.

---

## T6. Framework Leverage vs Custom Flexibility

**The choice**: minimal framework dependence. Bun.serve, Playwright, Anthropic SDK — nothing else at the framework level. No LangChain, no Pydantic, no agent framework.

**Evidence**: `package.json` has no workflow engine, no agent framework dependency.

**Benefits**:
- No upstream lock-in. If Bun disappears, most code is portable (TypeScript stays TypeScript).
- No framework-specific abstractions to learn.
- Features ship at gstack's pace, not at a framework's pace.

**Downsides**:
- Everything is built in-house: session-runner, touchfiles, eval-store, worktree manager, resolver pipeline.
- When a framework feature would have solved a problem (e.g., LangGraph for multi-agent), reinvention costs engineering time.
- New contributors learn gstack's conventions, not industry ones.

**When a different choice would be justified**: if the team ever wants to support non-Claude hosts as first-class (not just "generate skills for them") — i.e., if Codex users want the same runtime guarantees Claude users get — a framework with host abstractions would become attractive. For now, gstack is a Claude-first tool and the others are distribution targets.

---

## T7. Latency vs Completeness

**The choice**: complete, opinionated workflows that take minutes. No shortcut mode.

**Evidence**: `/ship` has 19 steps. `/autoplan` runs 4 review phases sequentially. `/investigate` has a 3-strike rule before escalation. ETHOS.md's "Boil the Lake" says: AI makes completeness cheap.

**Benefits**:
- Every ship has a CHANGELOG entry, a version bump, a canary check, a PR. No corner cuts.
- Each skill run is a full audit trail of what happened.
- Reduces "rushed shipping" class of bugs.

**Downsides**:
- Iteration loop on skills is slow. A small doc fix still goes through the full shipping workflow.
- Cost scales with completeness. A boil-the-lake `/ship` invocation burns more tokens than a minimum-viable one.
- Opinionation alienates users with different workflows.

**When a different choice would be justified**: for contributors working in fast edit-test cycles, a `/ship --quick` variant (skip canary, skip CHANGELOG summary rewrite) might be warranted. The current design deliberately avoids it to prevent drift.

---

## T8. Cost vs Accuracy

**The choice**: ensemble ML + deterministic tripwire; per-call LLM judge with single retry; dynamic cost accounting.

**Evidence**: `combineVerdict` requires 2-of-N ML classifiers at WARN; canary always BLOCKs; `llm-judge.ts` retries once on 429.

**Benefits**:
- Ensemble rule kills the Stack Overflow FP class without expensive manual tuning.
- Pricing is computed per-run — you can see exactly what an eval cost.

**Downsides**:
- LLM judge has no temperature pinning; scores drift run-to-run.
- Single-retry on 429 is brittle; sustained rate-limits just fail.
- No budget kill switch; cost overruns are possible.

**When a different choice would be justified**: every time the team uses eval scores to make a shipping decision, they should re-examine variance. Running judge at `temperature: 0` with structured output would cost nothing extra and remove a source of flakiness.

---

## T9. Developer Productivity vs Operational Rigor

**The choice**: ship fast, document norms in CLAUDE.md, enforce mechanically where it's cheap.

**Evidence**: CLAUDE.md is 500+ lines of rules, many of which are enforced by the skills themselves rather than CI. Binary-in-git is addressed by "never `git add .`" (convention) rather than a pre-commit hook (mechanism). Merge conflicts on SKILL.md are addressed by "regenerate from tmpl" (convention).

**Benefits**:
- Contributors get a clear philosophy document, not a pile of lint rules.
- Changes that would be mechanically blocked elsewhere (e.g., committing a binary by accident) are caught by the norm + code review.
- Operational overhead stays small.

**Downsides**:
- Every convention is violable. The binary-in-git story has lasted years despite "never do this" in docs.
- New contributors must read and internalize before shipping.
- Onboarding scales linearly with rule count.

**When a different choice would be justified**: as the contributor base grows, mechanical enforcement becomes the path of lower friction. The first things to mechanize: dist binary check, import-boundary check, CHANGELOG entry presence, skill-generation freshness (already CI'd).

---

## T10. Portability vs Opinionated Host Support

**The choice**: multi-host skill distribution (10 hosts supported), but only Claude is first-class.

**Evidence**: `hosts/*.ts` are 10 host configs, but the preamble, the `$B` alias, and the Agent tool are Claude-specific. Other hosts get generated skill content with path rewrites and tool renames but not feature parity.

**Benefits**:
- Market reach: Codex / Factory / OpenCode users can benefit from the best of gstack's skill content.
- Skill templates stay portable at the content level.

**Downsides**:
- Runtime parity is not guaranteed. A Codex user won't get the sidebar, the browse daemon is not wired up, evals don't run.
- Multi-host generation adds 10x output files + 10x places a skill can look wrong.
- Testing is Claude-only; other hosts rely on manual verification.

**When a different choice would be justified**: if a non-Claude host becomes a serious user base, the team should either (a) invest in full runtime parity for that host, or (b) stop generating skills for hosts that can't consume them meaningfully. Today it's a "no regrets" hedge, but the cost is review surface area.

---

## T11. Observability vs Overhead

**The choice**: minimal observability. stdout logs, JSONL attack log, JSON eval store. No structured logger, no tracing, no APM.

**Evidence**: no OpenTelemetry / pino / winston in `package.json`.

**Benefits**:
- Every log is greppable with Unix tools.
- No tracing backend to run.
- Fast.

**Downsides**:
- Cross-process traces (sidebar-agent + server) must be manually correlated.
- No built-in latency histograms, error rate dashboards, alerting.
- Production issues require a user to reproduce and capture logs themselves.

**When a different choice would be justified**: once gstack has enough users that the tail of "weird production issues" has a daily volume, a lightweight structured logger (like pino) with a local log file + an opt-in upload would cost little and return a lot. Today the user base is tight enough that direct contact resolves most issues.

---

## T12. Security Strictness vs User Escape Hatches

**The choice**: defense-in-depth for pages, but lots of user escape hatches (`GSTACK_SECURITY_OFF=1`, `GSTACK_SECURITY_ENSEMBLE=deberta`, kill-switches, bypass envs).

**Benefits**:
- Power users can unblock themselves during an FP.
- Research modes (DeBERTa ensemble) can be toggled without code changes.
- "Security that blocks legitimate work" is less likely to stick around.

**Downsides**:
- A compromised user env (`GSTACK_SECURITY_OFF=1` set by malicious installer) disables defenses silently.
- No audit log of who set which env var when.
- Feature flags become load-bearing and hard to retire.

**When a different choice would be justified**: for fleet deployments (Garry's YC portfolio companies rolling gstack out team-wide), env-var kill switches should require corporate-policy confirmation. Single-user installs are fine as-is.

---

## Meta-Observation on the Tradeoff Pattern

Most of gstack's decisions cluster around one axis: **code does the mechanical, unambiguous work; prompts do the judgmental, contextual work; documentation does the cultural work**. This is a coherent philosophy that yields many of the specific trade-offs above.

- Code is fast and predictable → used for auth, state, allowlists, compilation.
- Prompts are flexible and context-aware → used for severity, review, orchestration.
- Docs are cheap and human-readable → used for conventions, invariants, philosophy.

Where it breaks down: when a documentation-level invariant is load-bearing (the module boundary constraints), the design is fragile. These are the cases that would most benefit from mechanical enforcement without hurting the elegant overall architecture.
