# _phase2-study-pack.md

> Compact study pack distilled from the second pass. Items are ranked by "how often you'll need to reach for this under interview pressure" and "how load-bearing this is for architectural reasoning about gstack."

---

## Top 20 Concepts To Explain Out Loud

Rehearse each until you can deliver a 60-90-second explanation from cold memory.

1. **Policy-vs-mechanism split**. Prompts make decisions; code enforces invariants; docs carry culture. This is the architectural organizing principle.
2. **Persistent daemon pattern**. Keep expensive initialization warm; pay HTTP cost per call. 2-3s → <100ms.
3. **Prompt compile pipeline**. Typed `Record<string, ResolverFn>` + regex expansion + fail-fast + per-host frontmatter transforms.
4. **Dual-listener topology**. Two `Bun.serve` ports. ngrok forwards the tunnel-only port. Allowlists at both path and command levels.
5. **Ensemble ML + deterministic canary**. Stochastic signals need a deterministic tripwire. Canary leak → BLOCK unconditionally. Ensemble requires 2-of-N at WARN.
6. **Two-process security architecture forced by tool constraint**. `@huggingface/transformers` can't `dlopen` from compiled Bun binary. The team split the ML work into a separate subprocess.
7. **Diff-based paid test selection**. Per-test touchfile declarations; `git diff` decides what runs; global touchfiles trigger everything; tier classification filters further.
8. **@ref abstraction for agent-facing locators**. Server-side locator map; agent uses short refs; never sees CSS/XPath.
9. **Preamble tier system**. Skills opt into behavioral contracts by tier (1-4). Each tier adds sections additively.
10. **Atomic state writes via .tmp + rename**. POSIX guarantee; no partial reads; `0o600` mode.
11. **Startup lock via `wx` file create**. Kernel-atomic; free; prevents concurrent daemon startup.
12. **Sidebar model routing by regex**. 15 lines decide Opus vs Sonnet. Zero inference cost, adequate accuracy for the intent split.
13. **Dynamic cost accounting**. `pricing.ts` × `resultLine.usage` = real dollars per run. Eval store captures it.
14. **Session runner for E2E tests**. `claude -p` subprocess with NDJSON streaming; no retry policy; timeout only.
15. **LLM judge reproducibility gap**. No temperature pin; regex JSON extraction. Scores drift.
16. **Tier classification (gate vs periodic)**. Gate runs per PR; periodic runs weekly. Filters orthogonally to diff selection.
17. **Boil the Lake philosophy**. AI makes completeness cheap. Opinionated complete workflows beat shortcuts.
18. **Evidence-before-action protocol (investigate)**. 3-strike rule; no fixes without root cause; freeze file enforces scope.
19. **Host config system**. 10 hosts, each with allowlist/denylist, description limits, conditional fields, path/tool rewrites.
20. **30-minute idle shutdown**. Exempts headed and tunnel modes; check cadence 60s; reset on every `/command`.

---

## Top 20 Follow-up Questions You Should Be Ready For

For each, have a specific answer grounded in gstack's code, not a generic one.

1. **Why a compile step for prompts instead of runtime interpretation?** → determinism, type safety, CI freshness gate, fail-fast.
2. **Why two ports instead of one with header-based origin check?** → headers are spoofable by any upstream proxy.
3. **Why ensemble ML + canary instead of one well-tuned classifier?** → single-classifier FPs (Stack Overflow prose) + deterministic attack signal coverage.
4. **Why files-and-grep for learnings, not vector DB?** → debuggability, zero infrastructure, privacy, adequate at current N.
5. **Why no retry on `claude -p` subprocess failures?** → fail fast, report error, let the human decide.
6. **Why 30-min idle timeout?** → memory/resource reclamation; balance between "skills resume" and "don't leak".
7. **Why `wx` flag for startup lock instead of a JSON-file-based lock?** → kernel atomicity; crash-safe; free.
8. **Why the Chrome extension has its own subprocess (sidebar-agent)?** → isolates ML classification from the compiled daemon binary.
9. **Why dynamic cost accounting instead of fixed per-test budget?** → visibility over approximation; real dollars per run.
10. **Why tier classification separate from touchfiles?** → gate-tier tests enforce invariants; periodic-tier is quality benchmarks. Different cadence.
11. **Why 10 hosts supported but only Claude first-class?** → generation is cheap; runtime parity is expensive; hedge without commitment.
12. **Why `temperature` not set on LLM judge?** → this is an oversight, not a deliberate choice. Should be `0`.
13. **Why no rate limit on `/command` HTTP?** → local-only, bearer-auth, single OS user. Tunnel surface is allowlisted.
14. **Why no CI for Windows?** → historical; CI runs Linux on Ubicloud. Node.js fallback exists but isn't validated mechanically.
15. **Why no checkpoint/resume for `/ship`?** → current gap. The design implicitly assumes skills finish within one Claude session.
16. **Why `security-classifier.ts` can't be imported from `server.ts`?** → Bun compiled binary extracts to temp dir; `@huggingface/transformers` v4 requires `onnxruntime-node` native module that fails `dlopen` from temp dir.
17. **Why no signed skill distribution?** → current gap. Supply-chain risk. Fleet rollout would require this.
18. **Why CHANGELOG is so large (321K) and committed?** → user-facing release notes, branch-scoped entries. Compresses to nothing in git.
19. **Why Bun.serve instead of Express/Hono/Fastify?** → no framework overhead; built-in; Bun is already the runtime.
20. **What breaks first when concurrent Claude sessions share a daemon?** → Chromium tab state (focus, scroll, cookies). No per-session context isolation.

---

## Top 10 Diagrams To Draw From Memory

Practice until you can whiteboard each in under 2 minutes.

1. **The skill compile pipeline**: `.tmpl` + resolvers + host configs → `gen-skill-docs` → per-host SKILL.md.
2. **The dual-listener topology**: client → cli.ts → server.ts with two Bun.serve rectangles, tunnel-only path+command allowlist.
3. **The daemon startup lifecycle**: readState → isHealthy → lock → spawn → health loop → runCommand.
4. **The security stack**: L1-L3 in server.ts, L4+L4b in sidebar-agent.ts, L5 canary in server.ts, L6 combineVerdict.
5. **The test tier selection flow**: git diff → touchfiles lookup → tier filter → EVALS_ALL override → test list.
6. **The process tree at runtime**: claude (Claude Code) → browse daemon → Chromium + sidebar-agent + (optional) ngrok.
7. **The `$B click @e3` request flow**: cli → HTTP POST → auth → resetIdleTimer → handler → browser-manager → locator map → Playwright.
8. **The preamble tier composition**: T1 core, T2 adds confusion+recovery+checkpoint, T3 adds repo-mode+search, T4 is passthrough.
9. **The host config transform**: input frontmatter + hostconfig → allowlist/denylist applied → extraFields added → conditional rules → output frontmatter.
10. **The eval result lifecycle**: session-runner → NDJSON → parseNDJSON → SkillTestResult → llm-judge → eval-store → compare (previous run).

---

## Top 10 Architectural Mistakes To Avoid In Interviews

### M1. Confusing Claude Code with gstack
gstack is NOT an agent framework. Claude Code is. If you catch yourself saying "gstack's agent loop" — correct it. Say "Claude Code's agent loop configured by gstack skills."

### M2. Listing six security layers without explaining the categories
L4 and L4b both run in one subprocess. L6 is not a defensive layer — it's a decision fusion step. Mumbling "six layers" signals you memorized rather than understood.

### M3. Claiming eval scores are comparable across runs
They're not — the judge is stochastic. Say "comparable in aggregate with variance" or "indicative, not reproducible."

### M4. Saying "cost is ~$4 per test run" as if it's a constant
It's a headroom budget mentioned in CLAUDE.md. Actual cost is computed per run. Get this right or you lose credibility.

### M5. Proposing a vector DB for learnings
The current design is deliberate. Before proposing embeddings, propose SQLite-FTS as an intermediate step. It preserves all the debuggability benefits.

### M6. Adding retries everywhere
gstack explicitly limits retries. `claude -p` has none. Startup health check has a time budget, not retries. Follow the pattern; don't reach for retries reflexively.

### M7. Proposing a framework migration (LangChain, LangGraph, AutoGen)
gstack's lack of framework dependency is deliberate. Proposing framework adoption without justifying why Claude Code's built-in loop is insufficient signals you're pattern-matching on buzzwords.

### M8. Missing the Windows fallback story
There's a whole second entrypoint (`server-node.mjs`) because Bun's Chromium pipe handling fails on Windows. A strong answer acknowledges platform-specific paths as architectural choices.

### M9. Treating the browse daemon as stateless
It's stateful (Chromium page objects, cookies, ref map, tab state). Treating it as stateless misses the singleton-concurrency risks.

### M10. Ignoring the "policy-in-prompts" framing
If you describe gstack as "a codebase with these features" without naming the prompt/code split, you've missed the architectural thesis. Lead with the split; everything else is detail.

---

## Top 10 Distinctions To Articulate Clearly

### D1. Skills (prompts) vs resolvers (code)
Skills are markdown templates that describe workflows. Resolvers are TypeScript functions that emit the shared sections skills include via `{{PLACEHOLDERS}}`.

### D2. `$B` (CLI alias) vs `browse` (binary) vs `browse daemon` (HTTP server)
`$B` is a shell alias. `browse` is the compiled binary. The daemon is the long-running HTTP server spawned by the binary when no server exists.

### D3. Browse commands (HTTP) vs Claude Code tools (Bash/Edit/Read/Write/Agent)
Browse commands are gstack-specific HTTP actions. Claude Code tools are the built-in tool surface Claude uses to call browse commands (via Bash) or write files.

### D4. Security layers L1-L3 (always on) vs L4-L4b (sidebar only)
L1-L3 are in-process pure-string transforms, run on every content path. L4-L4b are ML classifiers, run only in the sidebar-agent subprocess.

### D5. Canary (L5) vs content classifier (L4) vs transcript classifier (L4b)
Canary is deterministic string injection + detection. L4 classifies page content. L4b classifies conversation transcripts via Haiku.

### D6. Touchfiles (code-to-test mapping) vs E2E_TIERS (gate vs periodic)
Touchfiles determine WHAT runs given a diff. E2E_TIERS determines WHEN each test runs (every PR vs weekly).

### D7. Per-run cost (dynamic) vs per-test budget (fixed, in CLAUDE.md)
The documented budget ($4/gate run) is a ceiling. The real cost is computed per run from actual token usage.

### D8. Local listener vs tunnel listener
Two separate Bun.serve binds. Local is full-surface + bearer token. Tunnel is allowlisted paths + allowlisted commands + scoped tokens. ngrok forwards only the tunnel port.

### D9. Headed mode vs headless mode
Headed mode includes the Chrome extension's sidebar + sidebar-agent subprocess. Headless mode is CLI-only. Headed mode never idle-shuts-down.

### D10. Eval runs (`claude -p` tests) vs skill runs (user-facing workflows)
Eval runs are subprocess invocations that test skills. Skill runs are when a user types `/ship` in Claude Code and the skill executes. Evals are paid; skills are paid; these costs are tracked separately.

---

## Quick-Glance Cheat Sheet (for last 5 minutes before a session)

- **Philosophy**: Policy in prompts, mechanism in code, culture in docs.
- **Daemon**: Persistent, dual-port, atomic state, startup lock, 30-min idle.
- **Skills**: Compiled from typed resolvers, one-pass, fail-fast, 10-host output.
- **Security**: 3 always-on transforms + 2 subprocess ML + 1 canary + 1 combiner. 2-of-N at WARN → BLOCK. Canary → always BLOCK.
- **Tests**: Diff-select via touchfiles, tier filter (gate/periodic), dynamic cost per run.
- **Key files**: `browse/src/server.ts` (2835 lines), `browse/src/cli.ts` (1035), `scripts/gen-skill-docs.ts` (~600), `test/helpers/touchfiles.ts`, `test/helpers/pricing.ts`.
- **Key invariants**: `security-classifier.ts ⊄ server.ts`; `token-registry.ts ⊄ sse-session-cookie.ts`. Doc-enforced only.
- **Key gaps**: LLM judge reproducibility; per-PR cost cap; import-boundary tests; signed skill releases; checkpoint-resume for long skills.

---

## If You Can Only Remember Three Things

1. **Policy-vs-mechanism**: the architectural spine. Prompts make decisions; code enforces invariants. If you lose this, you lose everything.
2. **Two-process security**: forced by tool limitation, became an advantage. Fast path stays fast; ML cost is subprocess-isolated.
3. **Compile step for prompts**: single-pass typed expansion with fail-fast. Rare, elegant, transferable.

Everything else can be re-derived from these three.
