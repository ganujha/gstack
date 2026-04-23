# _cross-repo-comparison-v2.md

> This learning package covers one repository (gstack). The "cross-repo" lens here compares gstack's **internal architectural subsystems** at the level of choices and consequences, not features. Each subsystem can be thought of as a standalone mini-architecture with its own philosophy; reading them side-by-side reveals where gstack's principles cohere and where they diverge.
>
> The five subsystems compared:
> 1. **Browse daemon** (persistent HTTP + Chromium)
> 2. **Skill compile pipeline** (template → multi-host SKILL.md)
> 3. **Security stack** (two-process ensemble defense)
> 4. **Test infrastructure** (diff-selection + paid evals + worktree isolation)
> 5. **Chrome extension + sidebar-agent** (headed mode chat surface)

---

## 1. Architectural Philosophy

| Subsystem | Core philosophy | Where it shows |
|-----------|----------------|----------------|
| Browse daemon | "Keep the expensive thing warm; enforce invariants at transport layer." | Persistent process, atomic state, dual-listener port separation |
| Skill compile | "Compile decisions at build time; fail fast on drift." | Static registry, single-pass expansion, CI freshness check |
| Security stack | "Fast deterministic + slow probabilistic, in separate processes." | Split between `security.ts` (pure strings) and `security-classifier.ts` (ONNX subprocess) |
| Test infrastructure | "Pay only for what changed; quantify cost in dollars, not time." | Touchfiles, tier classification, dynamic pricing |
| Chrome extension + sidebar-agent | "Agent-in-the-browser; ML gatekeeper per conversation turn." | Sidebar subprocess; per-turn classification; SSE cookie session |

**Observation**: Subsystems 1, 2, and 4 share a "compile + enforce" philosophy — do the work up front, use runtime as cheap dispatch. Subsystems 3 and 5 share a "two processes, one policy" philosophy — force a process boundary to match a security or compilation boundary. The repo's architectural vocabulary is small: compile, enforce, separate.

---

## 2. Agent / Control Model

| Subsystem | Where decisions live | Who makes them |
|-----------|---------------------|----------------|
| Browse daemon | Skill templates (prompts) | Claude Code model |
| Skill compile | Template + host config (code) | Compile step at build time |
| Security stack | Thresholds (code) + canary (code) + ensemble rule (code) | Verdict combiner deterministically |
| Test infrastructure | Touchfiles + tier classification (code) | Shell `git diff` + static lookup |
| Chrome extension | Skill templates + sidebar routing regex (code) | Regex for cheap; model for content |

**Insight**: The browse daemon is the only "dumb I/O" subsystem — all its decisions are delegated to Claude. The other four subsystems encode decisions in code, with varying amounts of model involvement.

The architectural coherence: **code decides mechanical concerns; models decide contextual concerns**. Browse daemon = mechanical I/O. Security = mechanical enforcement of a policy ensemble. Tests = mechanical selection. Skill compile = mechanical output generation. Sidebar = mechanical routing + model execution.

---

## 3. Tool / Integration Model

| Subsystem | Tool surface | Abstraction strategy |
|-----------|-------------|---------------------|
| Browse daemon | HTTP POST `/command` with bearer token | Command registry + @refs hide locator details |
| Skill compile | `bun run gen:skill-docs` + resolver registry | Typed `Record<string, ResolverFn>` |
| Security stack | `classifyContent`, `checkTranscript`, `combineVerdict` | Pure functions returning verdict objects |
| Test infrastructure | `bun test`, `bun run eval:*`, `session-runner.ts` | Subprocess orchestration + NDJSON |
| Chrome extension | Sidebar ←→ server via HTTP + SSE + IPC | Cookie-based session auth |

**Insight**: Each subsystem has a single, narrow, typed interface. No subsystem exposes its internals to other subsystems except through the file system (state files, attack log, eval store). This is deliberate isolation.

---

## 4. Memory / State Model

| Subsystem | State locations | Durability |
|-----------|----------------|-----------|
| Browse daemon | `~/.gstack/browse.json`, `browse.json.lock`, Chromium user-data | Session |
| Skill compile | None at runtime; `jargon-list.json` + resolver functions at build time | Build-time only |
| Security stack | `~/.gstack/security/session-state.json`, `attempts.jsonl`, `device-salt`, `~/.gstack/models/` | Mix: session, persistent, persistent, persistent |
| Test infrastructure | `~/.gstack/projects/$SLUG/evals/`, worktree dedup index | Persistent |
| Chrome extension | Local storage in extension + cookies + session-state.json | Session |

**Insight**: gstack's state is **explicit and inspectable**. Every subsystem's state is a file you can `cat`. No databases. No caches you can't introspect. The closest thing to a cache is the ML model directory, and it's just files.

This is a deliberate choice with a clear cost (no semantic retrieval; no centralized fleet dashboard) and a clear benefit (everything is debuggable by Unix tools).

---

## 5. Workflow and Developer Ergonomics

| Subsystem | Developer touchpoints | Iteration speed |
|-----------|---------------------|-----------------|
| Browse daemon | Edit `.ts`, run `bun run build`, reattach | Medium (compile step + restart) |
| Skill compile | Edit `.tmpl`, `bun run gen:skill-docs`, commit both | Fast (seconds) |
| Security stack | Edit `security.ts` and/or `security-classifier.ts`; rebuild | Slow (two processes, one compiled) |
| Test infrastructure | Add test, declare touchfiles, run locally | Medium (evals are expensive) |
| Chrome extension | Edit `sidepanel.js`, reload extension | Fast (live reload) |

**Observation**: The hardest subsystem to iterate on is security (two processes, one ONNX-dependent). The easiest is skill compile (single `bun run` rebuild). This shapes contributor behavior: skill templates see daily churn; security code sees careful, infrequent edits.

---

## 6. Runtime Model

| Subsystem | Runtime model | Process count |
|-----------|--------------|---------------|
| Browse daemon | Long-lived Bun process with Playwright subprocess | 2 (daemon + Chromium) |
| Skill compile | Short-lived build step | 1 (ephemeral) |
| Security stack | Split: in-process fast path (in daemon) + subprocess slow path (sidebar-agent) | +1 sidebar-agent (when headed) |
| Test infrastructure | Short-lived `claude -p` subprocesses + LLM API calls | N per test |
| Chrome extension | Browser runtime + MV3 service worker + content scripts | Browser-internal |

**Insight**: The total process count is intentionally small in normal use (daemon + Chromium), growing only when headed mode or tests are active. This keeps the "what's running on my machine" story simple.

---

## 7. Safety / Reliability Controls

| Subsystem | Safety mechanisms | Gaps |
|-----------|------------------|------|
| Browse daemon | Atomic state write, startup lock, bearer auth, idle shutdown, dual-listener allowlists | No checkpoint for long skills; single Chromium singleton |
| Skill compile | Type system, fail-fast, token ceiling warning, CI freshness | Literal placeholder regression if regenerated incorrectly; no import-boundary tests |
| Security stack | Ensemble rule, deterministic canary, kill switch, attack log rotation | No red-team fixtures; prompt-only "boundary" for Codex; module invariants doc-only |
| Test infrastructure | Worktree isolation, dedup index, NDJSON parse, 429 retry | No cost cap; no reproducible judge (temperature unpinned) |
| Chrome extension | SSE session cookie, sidebar L4/L4b gate | Extension-specific security (MV3 CSP) not audited here |

**Cross-cutting gap**: documentation-enforced invariants appear in both Security and Skill compile. Neither subsystem has mechanical enforcement of the load-bearing boundaries.

---

## 8. Observability Maturity

| Subsystem | What you can see | What's missing |
|-----------|-----------------|----------------|
| Browse daemon | `/health` endpoint, stdout logs | No trace IDs, no request latency histograms |
| Skill compile | stdout warnings, CI failure logs | No token budget dashboard, no regression reports |
| Security stack | Attack log (JSONL, rotated) | No trend analysis, no per-domain breakdown |
| Test infrastructure | Eval store with trend comparison | No cross-PR cost aggregation, no per-skill flakiness score |
| Chrome extension | Browser devtools + SSE | No extension-specific metrics |

**Observation**: Test infrastructure has the best observability (structured eval store with explicit trend logic). Browse daemon has the weakest (stdout only). A production-grade upgrade would start with daemon request logging and skill-compile token budget tracking.

---

## 9. Production Readiness

| Subsystem | Single-user | Small team | Fleet |
|-----------|------------|------------|-------|
| Browse daemon | High | Medium | Low-Medium |
| Skill compile | High | High | High |
| Security stack | High | High | Medium |
| Test infrastructure | High | Medium | Low |
| Chrome extension | Medium | Medium | Low |

**Why skill compile is the most production-ready**: it has no runtime, no state, and its failure mode is a build error you can't miss. Deterministic, typed, inspectable.

**Why test infrastructure degrades at fleet scale**: cost accounting is per-run, not cross-PR; no centralized dashboard; eval store is local file pile.

**Why extension is the weakest fleet case**: MV3 policy, cross-origin constraints, and user-install complexity scale poorly.

---

## 10. Extensibility

| Subsystem | What's easy to extend | What's hard |
|-----------|----------------------|-------------|
| Browse daemon | Add a new browser command (`commands.ts` + handler) | Add a third listener or a new surface type |
| Skill compile | Add a new resolver or a new host | Add nested placeholder support (would require re-design) |
| Security stack | Tune thresholds via env var | Add a new classifier ensemble member (requires subprocess coordination) |
| Test infrastructure | Add a new skill test | Add a cross-project comparison (no central store) |
| Chrome extension | Add a new sidepanel feature | Add cross-extension coordination |

**Insight**: extensibility is shallow (easy local additions) and deep (hard architectural changes). This is typical of cohesive designs. The places where deep extensibility would be needed (cross-repo evaluation, multi-ensemble classifier, cross-user fleet dashboards) are also the places where gstack's current architecture shows its single-user-first opinion.

---

## 11. Best Learning Value

For a reader who has ~2 hours and wants to learn from gstack's architecture:

| Subsystem | What you learn | Why unique |
|-----------|---------------|-----------|
| Skill compile pipeline | Prompt engineering at scale with types | Rare in public codebases |
| Browse daemon | Persistent tool-server pattern for AI agents | Directly transferable |
| Security stack | Two-process ML + deterministic tripwire | Rare real-world multi-layer defense |
| Test infrastructure | Diff-based paid eval selection | Novel pattern |
| Chrome extension | Headed AI agent in browser UI | Less architectural insight |

**Highest-value** subsystems for learning: skill compile (rare pattern, simple code, clear philosophy) and security stack (real threat model, concrete mitigations).

---

## 12. Best Interview Value

| Subsystem | Best questions it enables | Why strong |
|-----------|---------------------------|-----------|
| Browse daemon | "Design a persistent daemon for an AI tool" | Concrete system design target |
| Skill compile | "Design a prompt compile pipeline" | Few candidates have seen this pattern |
| Security stack | "Design prompt injection defense" | Real threat + real mitigations + trade-offs |
| Test infrastructure | "Design cost-aware paid test selection" | Tests business judgment not just engineering |
| Chrome extension | "Route messages between cheap and expensive models" | Shows simplicity thinking (regex vs classifier) |

**Strongest interview questions** come from security and skill-compile subsystems — they hit problems candidates rarely see and have concrete, non-obvious answers grounded in gstack's code.

---

## Summary: Architectural Coherence Report

gstack's five internal subsystems share three architectural moves consistently:

1. **Compile, don't interpret** — put decisions in build-time code where possible (skill compile, touchfiles, host config). Don't defer decisions to runtime when a compile step can make them.
2. **Separate fast from slow** — physical boundaries between hot paths (`security.ts` in-process) and cold paths (`security-classifier.ts` subprocess); tier classification for tests; cheap regex before expensive model.
3. **Enforce at transport layer** — auth at HTTP entry, not inside handlers; path allowlist before route dispatch; canary check before content leaves untrusted context.

Where the coherence breaks down: **documentation-enforced invariants** (module boundaries, "don't git add .", "don't claim pre-existing failure") appear in multiple subsystems and are all the places where second-pass critique finds the sharpest reliability risks. A natural next investment for gstack would be moving the most load-bearing of these from docs into code.
