# 19-teach-back-pack.md — gstack

> Teach-back material at three lengths and three audiences. Each explanation is self-contained — no prerequisite reading.

---

## The 2-Minute Explanation

gstack is an engineering productivity tool that turns Claude Code into a shipping machine. It does three things. First, it gives Claude 40 ready-made workflow recipes ("skills") — how to ship a feature, how to review a PR, how to investigate a bug — written as markdown prompts compiled from typed TypeScript templates. Second, it runs a persistent browser daemon so the AI can drive a real Chromium without paying a 3-second cold start on every click. Third, it has a six-signal prompt-injection defense layered between the browser and the AI so that hostile web pages can't hijack the agent. You install it with one command. It works with any Claude Code session on any project. Garry Tan built it.

---

## The 5-Minute Explanation

gstack lives at the intersection of prompt engineering, browser automation, and AI agent tooling. The core insight is that policy should live in prompts and mechanism should live in code, and the two should meet through a compile step.

The **skill system** is 40+ hand-authored markdown templates, each describing a full engineering workflow (ship a feature, review a PR, investigate a failure, run office hours). A TypeScript compiler expands `{{PLACEHOLDERS}}` into final prompts using a static registry of typed resolver functions. The same template generates skills for 10 different AI hosts (Claude, Codex, Factory, OpenCode, ...) with per-host frontmatter transforms and path rewrites. A CI check fails if any generated skill drifts from its template.

The **browse daemon** is a Bun HTTP server that keeps Chromium warm. The CLI (`$B click @e3`) connects to a locally running server via bearer-token-authenticated HTTP; startup is atomic-locked; state persists in `~/.gstack/browse.json`; idle shutdown reclaims memory after 30 minutes. The daemon binds two separate TCP ports: a local listener for full command access and a tunnel listener (only paths: `/connect`, `/command`, `/sidebar-chat`; only commands: 17 browser-driving verbs) that ngrok forwards. Security is physical port separation, not header inspection.

The **security stack** gates untrusted page content through layered signals. Fast, deterministic transforms (datamarking, hidden-element stripping, URL filtering) run in every path. Two ML classifiers (TestSavantAI ONNX + Claude Haiku transcript) run in a separate `sidebar-agent` subprocess because `@huggingface/transformers` can't `dlopen` ONNX from a compiled Bun binary. A deterministic canary injection is the tripwire — if the canary escapes, something exfiltrated it. A verdict combiner requires 2-of-N ML classifiers to agree at WARN threshold before escalating to BLOCK.

The **test infrastructure** runs in three tiers: free static validation (<2s, always runs), paid LLM-as-judge quality evals (~$0.15/run, diff-selected), and paid end-to-end `claude -p` subprocess tests (~$3.85/run, diff-selected). A touchfiles table declares per-test source dependencies; global touchfiles (infra changes) trigger all tests. Results persist to `~/.gstack/projects/$SLUG/evals/` with trend comparison.

The **philosophy**, captured in ETHOS.md, is "Boil the Lake" — AI makes completeness cheap, so always do the full thing rather than shortcuts. This is why `/ship` is 19 steps, why `/autoplan` runs four review phases sequentially, why the codebase prefers deterministic enforcement for mechanical work and model discretion for judgment.

What's novel: the compile step for prompts, the two-process security split, the dual-listener topology, and the use of a 15-line regex for Opus-vs-Sonnet sidebar routing where many agent systems would reach for a classifier.

---

## The 15-Minute Whiteboard Explanation

**Opening frame.** gstack extends Claude Code with skills, a persistent browser, layered security, and a paid test harness. It is NOT an agent framework. Claude Code's built-in agent loop is the runtime; gstack is the configuration and the tool library.

### Layer 1 — The skill system (5 minutes of the 15)

Draw the compile pipeline:
```
SKILL.md.tmpl ──┐
resolvers/*.ts ──┼─► gen-skill-docs.ts ─► SKILL.md (×10 hosts)
hosts/*.ts     ──┘
```

Key facts to communicate:
- Templates contain `{{PLACEHOLDERS}}`.
- `scripts/resolvers/index.ts` is a static `Record<string, ResolverFn>` — no plugins, no dynamic discovery.
- Placeholders expand in a single regex pass; any unresolved token at the end throws a build error.
- The preamble is composed by tier (1-4), each tier additively including more behavioral sections (confusion protocol, context recovery, continuous checkpoint).
- Per-host frontmatter transforms handle allowlist/denylist modes, description length limits, conditional fields, field rename.
- A dry-run regeneration + diff is the CI freshness check.

Key design decision: **compile step for prompts**. Not a macro system. Not a plugin system. Just a typed string interpolator with fail-fast.

### Layer 2 — The browse daemon (5 minutes)

Draw the dual-listener topology:
```
$B command
    │
    ▼
cli.ts ──(HTTP + bearer token)──► server.ts ──► Playwright ──► Chromium
                                     ▲
                                     │
                          ngrok ◄─── tunnel port (allowlisted paths + commands)
```

Key facts:
- Atomic state file `~/.gstack/browse.json`; startup lock via `wx` flag.
- Bearer token issued at startup; tunnel-scoped tokens issued separately via a pairing ceremony.
- `$B` commands use `@refs` (like `@e3`) not CSS selectors — the server maintains the ref→locator map.
- Idle shutdown after 30 minutes, exempt for headed and tunnel modes.
- Windows uses a Node.js fallback (`server-node.mjs`) because Bun's pipe handling for Chromium doesn't work there.
- Sidebar model routing (Opus vs Sonnet) is a 15-line regex: analysis words → Opus; short imperative action → Sonnet; default Opus.

Key design decision: **mechanism vs policy split**. The daemon has zero policy in it. It enforces auth, transport invariants, state atomicity. Everything else is in prompts.

### Layer 3 — The security stack (3 minutes)

Draw the two-process split:
```
server.ts (compiled binary, pure strings)
    │
    ├─ L1 datamarking      (always on)
    ├─ L2 hidden strip      (always on)
    ├─ L3 URL filter        (always on)
    ├─ L5 canary check      (deterministic tripwire)
    └─ L6 combineVerdict    (consumes signals)
         ▲
         │ session-state.json (IPC)
         │
sidebar-agent.ts (non-compiled, subprocess, ML)
    ├─ L4  classifyContent  (TestSavantAI ONNX)
    └─ L4b checkTranscript  (Claude Haiku)
```

Key facts:
- The ONNX classifier cannot be imported from the compiled browse binary (tool limitation). This forces the two-process architecture.
- `combineVerdict` requires 2-of-N ML classifiers to agree at WARN (0.60) before escalating to BLOCK.
- Canary leak is always BLOCK, deterministically.
- `GSTACK_SECURITY_OFF=1` kills ML layers but keeps canary.

Key design decision: **fast deterministic + slow ML, in separate processes**. Most pages never pay ML cost; the ones that do can tolerate it because they're part of a chat conversation.

### Layer 4 — The test harness (2 minutes)

Draw the tiers:
```
Tier 1: bun test              — free, <2s, always runs
Tier 2: test:evals            — paid, diff-selected, LLM judge
Tier 2: test:e2e              — paid, diff-selected, claude -p subprocess
Tier 3: periodic              — weekly cron, full suite
```

Key facts:
- `touchfiles.ts` declares each test's file dependencies.
- Global touchfiles (`session-runner.ts`, `eval-store.ts`, itself) trigger all tests.
- `E2E_TIERS` classifies each test as `gate` or `periodic`.
- Cost is computed per run from actual token usage × model-specific pricing; it is not a fixed per-test budget.
- LLM judge uses `claude-sonnet-4-6`; does NOT pin temperature; extracts JSON with regex.

Key design decision: **pay only for what changed**. Diff-selection plus tier filtering keeps costs sub-$10 per PR on most changes.

---

## Teach This To a Strong Engineer

Frame: "This is an agent tooling codebase with an unusual architectural split — prompts make decisions, code enforces invariants. The interesting stuff is in how the compile step, the daemon's process model, and the security stack all use that split."

Walk them through:
1. The resolver registry (`scripts/resolvers/index.ts`) — why it's static.
2. The two `Bun.serve` calls in `server.ts` — why physical port separation beats header checks.
3. The `security-classifier.ts` module-top comment — why the second process exists.
4. The `pickSidebarModel` function — why a regex is the right choice here.
5. `test/helpers/pricing.ts` — why cost is dynamic and what this tells you about the eval system.

What they should take away: **the repo is small because most of the "logic" is in markdown prompts**. This is an architectural choice that changes what kind of bugs you get and how you fix them.

What they should debate: whether prompt-only invariants (Codex don't-read-`~/.claude`, "pre-existing failure" claims) are robust enough. The answer is "no, and the team knows — mechanical enforcement is the ongoing work item."

---

## Teach This To a Staff Engineer

Frame: "gstack is a study in cultural-plus-mechanical architecture. The code enforces invariants where cheap; the prompts enforce conventions where necessary; the docs enforce culture where nothing else works. Your interest should be the boundary between these."

Key discussion points:
1. **Load-bearing documentation**: the two module-boundary invariants that aren't lint-enforced. These are reliability risks with known mitigations.
2. **Stochastic judge, deterministic consumer**: eval trends are treated as comparable across runs, but the judge is un-pinned. This is a latent issue that compounds over time.
3. **Singleton concurrency**: the Chromium-per-user design was correct when shipped; as concurrent skill sessions become normal, per-session contexts become necessary.
4. **Multi-host risk surface**: 10 hosts generated from one template. Claude is tested; the others are not. This is a compounding divergence problem.
5. **Binary-in-git debt**: CLAUDE.md calls it out; nothing enforces it. Small but illustrative of the culture-vs-mechanism gap.

What they should notice: the architecture is more sophisticated than it looks because the boundaries are enforced in different ways in different places. The places where documentation is the enforcer are the places to watch.

What they should ask: what would it take to get to 80% mechanical enforcement of the documented invariants? Answer: import-boundary tests, pre-commit hook for `dist/`, `temperature: 0` + structured judge output, red-team test fixtures. These are 1-3 day items each. Not hard; just not yet done.

---

## Teach This To a CTO

Frame: "gstack is a working implementation of 'AI makes completeness cheap.' It uses Claude as the agent runtime and provides everything else — prompts, browser automation, security, tests. The investment thesis is that your team produces more, more consistently, with fewer shortcuts, because the skill system makes the long version of every workflow the easy path."

Executive-level talking points:
1. **Effort compression**: the repo claims ~3-100x compression depending on task. `/ship` (19 steps, 30 minutes) replaces 2 engineer-hours. `/review` replaces a half-day PR review process. `/autoplan` replaces a week of design-and-review.
2. **Cost profile**: paid evals ~$4-8 per PR; local daemon is free. Predictable; diff-selection keeps routine PRs under $1.
3. **Security posture**: a real multi-layer defense for a real threat (prompt injection from web pages). Single-user install is defensible today; fleet deployment needs signed releases and cost dashboards.
4. **Vendor exposure**: Claude Code is the runtime; Anthropic API for judge and sidebar; HuggingFace for classifier models; Supabase for telemetry. Bun for build. No lock-in beyond Claude Code itself.
5. **Operational readiness**: single-user ready; team-ready with some caveats; not fleet-ready (see 16-production-readiness-review.md top-10 upgrades).

What you should ask before adopting:
- Does your team tolerate the opinionation? (40+ complete workflows, not configurable.)
- Is Claude Code your primary AI assistant? (Other hosts are second-class.)
- Do you have eval budget? ($50-200/week during active use is realistic.)
- Who owns the daemon on shared dev machines? (Single-user per machine today.)

What it replaces: no tool; it slots alongside your existing CI, review process, and shipping tooling. It does not replace `git`, GitHub, or your test framework. It provides opinionated agent workflows that call those tools.

---

## 10 Most Important Takeaways

1. Policy is in prompts; mechanism is in code. This split is the architectural core.
2. The skill system is a **compile pipeline** (typed resolvers, static registry, fail-fast), not a macro system or plugin framework.
3. The browse daemon is a **persistent HTTP server** with physical dual-listener port separation, not a header-based allowlist.
4. The security stack is **two-tiered** — fast always-on transforms in-process; slow ML signals in a subprocess — and the split is forced by a tool limitation that became an architectural advantage.
5. **`combineVerdict` ensemble + deterministic canary** is the attack-defense core. Canary always wins; ML requires 2-of-N agreement.
6. **`@ref`-based browser automation** eliminates a class of agent errors by hiding selectors behind short server-side references.
7. **Diff-based test selection** with touchfiles keeps paid eval cost under $10/PR on most changes.
8. **Cost is computed dynamically** from token usage × model prices; the eval store tracks real dollars per run.
9. **Sidebar model routing is a 15-line regex** — a strong argument for simplicity over classifier-everything.
10. The **"Boil the Lake"** philosophy (AI makes completeness cheap) is why skills are 19 steps, not 3.

---

## 5 Biggest Misconceptions To Avoid

1. **"It's an agent framework."** No. Claude Code is the agent framework. gstack is a configuration library and a tool library.
2. **"The security stack has six independent layers."** Closer to three always-on transforms + two subprocess ML signals + one deterministic tripwire + one verdict combiner. Counting "6" is a mnemonic.
3. **"The LLM judge gives comparable scores across runs."** No — temperature is unpinned; JSON extraction is regex-based. Scores drift.
4. **"Cost is ~$4 per run."** That's a headroom budget. Actual cost is computed per run and depends on token usage.
5. **"Multi-host means multi-runtime."** No — only Claude is first-class at runtime. Other hosts get generated skill content but not daemon access, not evals, not sidebar.

---

## 5 Strongest Interview Stories Enabled By This Repo

### Story 1 — Persistent daemon cut per-command latency 20x
"We moved browser automation from stateless `npx playwright each call` to a persistent Bun HTTP server. Cold-start 2-3s per command became warm <100ms. 20-command workflows went from 40 seconds of pure overhead to near zero. The architecture: atomic state file, startup lock, idle shutdown, bearer-token auth, dual-listener port separation for remote access. Single-binary deploy via `bun build --compile`."

### Story 2 — Ensemble ML plus deterministic canary killed a whole FP class
"Stack Overflow pages — which literally contain 'ignore previous instructions and do X' as prose — were triggering our prompt-injection classifier. Single-classifier confidence scores couldn't distinguish instructional writing from injection. We moved to an ensemble rule: BLOCK only when 2-of-N classifiers agree at WARN threshold. Canary injection is the deterministic tripwire — if the string escapes, BLOCK unconditionally. Stack Overflow stopped getting blocked; real exfiltration still caught."

### Story 3 — Compile step for prompts
"We had 40+ skills across 10 AI hosts. Copy-paste of preamble sections was drifting. We built a typed resolver registry — `Record<string, ResolverFn>` — plus a template language with `{{PLACEHOLDER}}` expansion. One template, ten hosts, fail-fast on missing resolvers, CI freshness check on generated files. Adding a new behavioral instruction to all tier-2+ skills is now one file change."

### Story 4 — Diff-based eval selection cut paid test cost
"Our E2E suite cost ~$4/run. Running everything per PR was unsustainable. We declared per-test dependencies in a touchfiles table: `test X depends on files [a.ts, b.ts]`. `git diff --name-only` decides what runs. Global touchfiles (session-runner, eval-store, touchfiles itself) trigger everything. Typical PR cost dropped to $0-2; full-suite runs still available via `EVALS_ALL=1`."

### Story 5 — Cost routing with regex, not a classifier
"Our sidebar chat needed to route between Opus (thorough) and Sonnet (cheap, fast). Opus is 5-10x the cost and latency with zero quality improvement on 'click @e3'. We looked at training an intent classifier. Instead, we wrote 15 lines of regex: analysis verbs → Opus; short imperatives → Sonnet; default Opus on ambiguity. Zero inference cost, zero maintenance burden, as accurate as a classifier for our intent split."
