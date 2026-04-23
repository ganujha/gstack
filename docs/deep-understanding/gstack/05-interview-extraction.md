# 05-interview-extraction.md — gstack

## What Is Interview-Worthy in This Repo

- **Persistent browser-as-a-service daemon**: architectural decision with measurable latency impact
- **Prompt compilation pipeline**: TypeScript resolvers → multi-host markdown output
- **6-layer ML security stack**: ensemble classifiers + deterministic canary
- **Dual-listener port-separation security**: physical topology vs. header-based controls
- **Diff-based test selection at ~$4/run cost**: pragmatic cost engineering
- **LLM-as-judge evaluation system**: automated quality scoring with trend tracking
- **Git worktree isolation for test parallelism**
- **Preamble tier system**: behavioral contracts by skill complexity
- **Host abstraction**: one template generates prompts for 10 AI agents
- **Atomic server startup with `wx` file lock**: TOCTOU prevention

---

## 10 System Design Questions

### 1. "Design a browser automation tool for AI agents with sub-second latency."
**Answer bullets:**
- Cold-start Chromium takes 2-3s; persistent daemon reduces per-command cost to ~100ms
- HTTP server (Bun.serve) accepts commands as POST requests with JSON args
- State machine: `readState()` → health check → start if unhealthy → run command
- Lock prevents concurrent startup races (atomic `wx` open)
- Session state (cookies, tabs) persists across agent turns
- Version hash detects stale server; auto-restart on mismatch
- Dual-listener for local vs. tunnel access (physical port separation > header checks)

### 2. "How would you design a prompt distribution system for multiple AI agents?"
**Answer bullets:**
- Single source of truth: `.tmpl` files with `{{PLACEHOLDER}}` syntax
- Build-time expansion via typed TypeScript resolver functions (type safety at generation)
- Per-host config: frontmatter transforms, path rewrites, tool renames (HostConfig interface)
- Host-specific output directories: `.claude/skills/`, `.agents/skills/`, `.factory/skills/`
- Validation: CI checks that generated files are fresh vs. template
- Token budget monitoring: warn if generated prompt exceeds 160KB
- Freshness gate: `skill-docs.yml` workflow blocks merges with stale generated files

### 3. "Design a layered security system for an AI agent that reads web content."
**Answer bullets:**
- Layer 1: Watermark/datamark injected text to detect exfiltration patterns
- Layer 2: Strip hidden DOM elements (opacity < 0.1, off-screen, color-matched)
- Layer 3: ARIA label injection pattern regex (known attack phrases)
- Layer 4: ML content classifier (TestSavantAI ONNX, 112MB model)
- Layer 4b: ML transcript classifier (Claude Haiku, uses full session context)
- Layer 5: Canary injection (deterministic tripwire, always overrides ML)
- Layer 6: Ensemble verdict (2-of-3 at WARN threshold → BLOCK)
- Key insight: single ML layer has FP risk; ensemble + deterministic prevents both FP and FN

### 4. "How do you evaluate the quality of AI-generated prompts at scale?"
**Answer bullets:**
- LLM-as-judge: use a capable model (Sonnet) to score output on rubric dimensions
- Rubric: clarity (≥4/5), completeness (≥3/5), actionability (≥4/5)
- Diff-based selection: only evaluate skills whose source files changed (cost control)
- Persist results: JSON store with schema version, per-project namespacing
- Trend comparison: `eval-compare.ts` auto-compares to previous run baseline
- E2E complement: `claude -p` subprocess runs the actual skill workflow (functional test)
- Three tiers: free validation (<2s), paid gate (~$0.15), paid E2E (~$3.85)

### 5. "Design a test infrastructure for an AI workflow system where tests cost money."
**Answer bullets:**
- Diff-based selection: `touchfiles.ts` maps source file → test dependencies (git diff driven)
- Two tiers: `gate` (CI-required, safety guardrails) vs. `periodic` (weekly cron, quality benchmarks)
- Cost estimation per test: declare `maxCostCents` in test definition
- Subprocess isolation: `claude -p` runs in independent process (no shared state)
- Worktree isolation: `lib/worktree.ts` creates fresh git worktrees per test (prevents state bleed)
- Result persistence: `eval-store.ts` with project-scoped namespacing
- Override: `EVALS_ALL=1` forces all tests (for periodic/emergency runs)

### 6. "How would you build a multi-agent review system for code quality?"
**Answer bullets:**
- Coordinator reads diff, classifies PR into review domains
- Specialist subagents: SQL safety, race conditions, LLM trust, shell injection, enum completeness
- Fanout strategy: parallel for independent reviewers, sequential for gating (CEO gates Design, etc.)
- Decision taxonomy: Mechanical (auto-fix), Taste (surface at gate), User Challenge (always ask)
- `AskUserQuestion` for ambiguous decisions: structured format with reasoning
- Anti-sycophancy rules: take positions, state evidence needed to change mind
- Fix-First: auto-fix obvious issues, batch-ask on complex ones

### 7. "Design a skill/plugin system for an AI coding assistant."
**Answer bullets:**
- Skills as markdown files in `~/.claude/skills/<name>/SKILL.md`
- Claude Code discovers skills via directory scan at session start
- Frontmatter: name, description, preamble-tier, allowed-tools, voice-triggers
- Voice triggers: natural language phrases that auto-route to the skill
- Generation: templates compiled to host-specific formats (Claude, Codex, Factory, etc.)
- Namespacing: `--prefix` mode uses `gstack-<skill>` names to avoid conflicts
- Symlink strategy: real directories with symlinked SKILL.md (enables live development)
- Health check: `skill:check` dashboard validates all installed skills

### 8. "How do you handle context management in a long-running AI workflow?"
**Answer bullets:**
- TodoWrite as in-context scratchpad: structured task state Claude can re-read
- Continuous checkpoint preamble: instruction to write progress every N steps
- Context recovery preamble: instruction to re-read TodoWrite if context was pruned
- Context save/restore skills: explicit file-based session bridging
- Confusion protocol: stop and ask user rather than hallucinate progress
- Practical limit: 42K-token skill + 200K context = ~80% used by prompt alone on large diffs
- Token optimization: prompt caching makes preamble cheaper across turns

### 9. "Design the state management for a CLI that wraps a long-running subprocess."
**Answer bullets:**
- State file (`browse.json`): pid, port, token, startedAt, serverPath, binaryVersion
- Health check over HTTP rather than process table (definitive liveness signal)
- Lock file with `wx` flag (atomic open, fails if exists = already starting)
- Binary version hash: detect stale server, auto-restart without user intervention
- Graceful shutdown: SIGTERM (2s wait) → SIGKILL; Windows: taskkill /T /F
- Port randomization (10000-60000): supports multiple concurrent workspaces (Conductor)
- Windows fallback: Node.js child_process (detached) because Bun pipe handling breaks Chromium

### 10. "How do you design a development workflow for AI-generated code at a startup?"
**Answer bullets:**
- ETHOS: "Boil the Lake" — AI makes completeness cheap; always do the full thing
- Effort compression: 3x-100x vs. human team (table in CLAUDE.md)
- Three-tier testing: free (<2s), gate (~$4), periodic (~$3.85)
- AI code quality scanner (slop-scan): catch genuine quality issues, not hide AI origin
- Bisect commits: every commit is a single logical change (independently bisectable)
- Community PR guardrails: never auto-accept changes to founder voice or philosophy
- CHANGELOG-as-product-notes: lead with user impact, not implementation details

---

## 10 Debugging / Reliability Questions

### 1. "The browse daemon stops responding mid-workflow. How do you diagnose?"
**Answer:**
- Check `~/.gstack/browse.json` — is pid still running? (`kill -0 <pid>`)
- Run `$B status` — does it return or timeout?
- Check `isServerHealthy()` — HTTP GET /health with 2s timeout
- If unhealthy: cli.ts auto-retries once on ECONNREFUSED, then gives up
- Root cause candidates: Chromium crash, Bun OOM, server binary version mismatch
- Fix: `$B stop` → `$B goto <url>` (triggers restart)

### 2. "An E2E eval is flaky — sometimes passes, sometimes times out. How do you investigate?"
**Answer:**
- Check `eval-store.ts` run history: is failure correlated with time (rate limits)?
- Check `session-runner.ts` `maxInterTurnMs`: is Claude taking longer between turns?
- Check context window: is the test fixture too large (CLAUDE.md warns against full SKILL.md copies)?
- Run in foreground (not background with `&` and tee) to see real-time output
- Check `llm-judge.ts` 429 retry logic: is it hitting Anthropic rate limits?
- Never kill and restart — `pkill` running eval processes loses results and wastes money

### 3. "Security classifier is blocking legitimate pages. How do you tune thresholds?"
**Answer:**
- Check `~/.gstack/security/attempts.jsonl` for recent blocks with domain + salted hash
- Current thresholds: BLOCK=0.85, WARN=0.60, LOG_ONLY=0.40
- Ensemble rule: 2-of-3 at WARN; single layer → WARN (Stack Overflow FP mitigation)
- Emergency bypass: `GSTACK_SECURITY_OFF=1` (canary still fires, ML skipped)
- Tune: the `WARN: 0.60` cross-confirm threshold is the FP control knob
- Validate: submit known-safe pages and known-attack pages, check verdict

### 4. "SKILL.md CI check fails saying generated file is stale. What happened?"
**Answer:**
- Someone edited `SKILL.md.tmpl` but didn't run `bun run gen:skill-docs`
- Or edited a resolver file (`scripts/resolvers/*.ts`) without regenerating
- Fix: `bun run gen:skill-docs` then `git add <skill>/SKILL.md`
- `skill-docs.yml` workflow reruns the generator and diffs output
- Check if `.tmpl` file has merge conflicts — generated SKILL.md conflicts must NEVER be resolved directly; resolve `.tmpl`, regenerate

### 5. "Browser @ref click fails silently after page navigation. Why?"
**Answer:**
- @refs expire when the snapshot's locator map becomes stale (page changed)
- After any navigation (`goto`, `click` that causes navigation), old @refs are invalid
- Fix: call `$B snapshot` again to get fresh @refs before the next write command
- Skills should re-snapshot after any navigation; this is a prompt instruction, not automatic

### 6. "The `./setup` script fails on macOS ARM64 during binary compilation. What do you check?"
**Answer:**
- codesign step: `codesign --remove-signature` then `codesign -s -` may fail if Xcode tools not installed
- Check `bun --version` — Bun >= 1.0.0 required
- Check if Playwright Chromium installation completed (`bunx playwright install`)
- Check disk space: browse binary is ~58MB
- Check if previous partial binary exists in `browse/dist/` — `rm browse/dist/browse` and retry

### 7. "Two Claude Code instances running the same skill corrupt shared files. How would you prevent this?"
**Answer:**
- Current state: no file-level locking on TODOS.md, CHANGELOG.md, VERSION
- git-level collision detection exists (merge conflicts on push)
- Short-term fix: write to temp file → atomic rename
- Better fix: use git worktrees for each session (lib/worktree.ts exists for tests)
- Detect: check for git lock file (`~/.../COMMIT_EDITMSG.lock`) before writing

### 8. "The LLM judge gives inconsistent scores across runs for the same content. How do you make evals reliable?"
**Answer:**
- Use low temperature (likely 0 or near-0 in llm-judge.ts, not verified)
- Use structured JSON output extraction (`callJudge<T>()` with schema)
- `EvalCollector` tracks trends — variance across runs is itself a signal
- Compare against baseline: `eval-compare.ts` auto-diffs
- For rubric dimensions, use numeric scales (1-5) not text labels — reduces interpretation variance
- Rate limit retry: 429 handling in llm-judge.ts prevents failed runs from skewing data

### 9. "The ngrok tunnel disconnects during a pair-agent session. What breaks?"
**Answer:**
- The remote agent's HTTP POST to the tunnel URL starts returning connection errors
- The local Chromium continues running unaffected (daemon is on local listener)
- No reconnect logic visible in evidence — likely requires `$B tunnel restart`
- State loss: remote agent loses context; must re-pair to get new setup key + instruction block
- Prevention: use ngrok paid plan (stable tunnels) vs. free (ephemeral)

### 10. "A new skill template produces `{{REVIEW_ARMY}}` literally in the output SKILL.md. How do you debug?"
**Answer:**
- `bun run gen:skill-docs --dry-run` — check output without writing files
- `bun test` → `skill-validation.test.ts` catches literal `{{` strings in generated files
- Root cause: either the resolver function threw, or the placeholder is not registered in `scripts/resolvers/index.ts`
- Check: `index.ts` resolver registry for the placeholder name (exact string match, case sensitive)
- Check: resolver function `return` value — if undefined/null/empty, placeholder may not be replaced

---

## 10 Architecture Trade-off Questions

### 1. "Why use a persistent daemon vs. per-command Chromium launch?"
- Persistent: 100ms/command, stateful sessions, ~200MB memory always
- Per-command: 2-3s cold start, no state, no memory overhead when idle
- gstack chose persistent: AI agents do multi-step workflows; cold start per step is unacceptable
- Trade-off: resource cost for performance and statefulness

### 2. "Why generate SKILL.md files from templates vs. runtime prompt assembly?"
- Generation: skill is a static file in Claude's skills directory; Claude reads it naturally; no runtime overhead
- Runtime assembly: requires a pre-processing step every session start; adds latency; harder to inspect
- gstack chose generation: SKILL.md is a Claude Code convention; static files work with prompt caching; easier to diff and review
- Trade-off: build step required; generated files can get out of sync with templates

### 3. "Why use Bun instead of Node.js as the runtime?"
- Bun: compiled binaries (~58MB), native TypeScript, native SQLite, Bun.serve built-in
- Node.js: wider ecosystem, more predictable edge cases, works without compilation
- gstack chose Bun for users (faster startup, no install friction), kept Node.js fallback for Windows (Bun pipe issues with Chromium)
- Trade-off: Bun-only bugs, Windows second-class support, 112MB+ model downloads

### 4. "Why use physical port separation for tunnel security vs. header-based origin checks?"
- Port separation: attacker with valid tunnel URL cannot reach bootstrap endpoints; unforgeable by design
- Header checks: can be spoofed by attacker who controls request headers (proxies, MITM)
- gstack chose port separation: higher security guarantee for lower implementation complexity
- Trade-off: two ports must be managed, two Bun.serve() instances

### 5. "Why use ARIA snapshot with @refs vs. XPath/CSS selectors for browser elements?"
- @refs: shorter in context, agent doesn't need to reason about selectors, stable within session
- XPath/CSS: longer strings, agent must generate valid selectors, fragile across page changes
- gstack chose @refs: AI context optimization; fewer tokens, fewer selector bugs
- Trade-off: @refs expire on navigation; requires re-snapshot protocol

### 6. "Why use `claude -p` subprocess for E2E tests vs. the Claude Agent SDK?"
- `claude -p` subprocess: tests real user-facing behavior (includes Claude Code's tool permissions, hooks, settings); fully isolated
- Agent SDK: easier to mock, faster, but tests a different code path than real users hit
- gstack chose `claude -p`: evaluating skills means evaluating the full Claude Code experience
- Trade-off: slower (30-45min for full suite), more expensive (~$3.85/run), harder to debug

### 7. "Why enforce preamble tiers vs. letting each skill include exactly what it needs?"
- Tiers: DRY, centralized updates, consistent behavioral baseline across skills
- Per-skill: maximum customization, no dependency on tier system, simpler to understand one skill in isolation
- gstack chose tiers: with 40+ skills, DRY wins; behavioral contract changes (context recovery, confusion protocol) propagate automatically
- Trade-off: skills lose visibility into what their preamble contains; bugs in one generator affect all skills at that tier

### 8. "Why use LLM-as-judge instead of human review for skill quality?"
- LLM judge: scalable, automated, 24/7, cheap per run, consistent rubric
- Human review: catches subtle nuance, judges real user value, no FP/FN from LLM hallucination
- gstack uses LLM judge as gate, human review for strategic decisions
- Trade-off: LLM judge can't evaluate "would a founder actually find this useful?" — it scores proxy metrics

### 9. "Why support 10 AI agent hosts vs. focusing on Claude only?"
- Multi-host: wider user base, defends against vendor lock-in, forces host-agnostic skill design
- Claude-only: simpler implementation, no risk of host-specific bugs, deeper Claude integration
- gstack supports multi-host: commercial reality (not all users are Claude Code users); strategic
- Trade-off: complexity in frontmatter transforms, path rewrites, tool renames; test coverage gaps for non-Claude hosts

### 10. "Why use diff-based test selection vs. always running all tests?"
- Diff-based: fast PR feedback (~$4 vs. potentially $40 for all tests), focused on changed code
- Always all: complete coverage guarantee, no touchfile maintenance burden
- gstack uses diff-based: at ~$8/PR (gate + E2E), "all tests always" would be prohibitive at volume
- Trade-off: touchfiles.ts must be kept accurate; missed dependency = missed test = silent regression

---

## 5 "What Would You Improve?" Prompts

### 1. "What would you change about the security architecture?"
**Angles:**
- Add user-visible security audit log (vs. just `attempts.jsonl`)
- Implement WebAssembly isolation for untrusted page script evaluation
- Add browser sandbox profiles per-domain (Chromium has this capability)
- Consider rate limiting at the command level, not just the tunnel denial level
- Test suite gap: no automated red-team tests for known injection payloads

### 2. "What would you change about the skill template system?"
**Angles:**
- Add resolver test coverage: unit tests for each resolver function's output
- Add placeholder type system: declare expected output format (prose, table, code block) per placeholder
- Add skill-level token budgets declared in frontmatter (not just a global warning)
- Consider lazy loading of large resolver sections (design.ts 58K, review.ts 54K) — only include relevant sections per skill

### 3. "What would you change about the context management model?"
**Angles:**
- Automatic context checkpointing (not prompt-instruction-based): write TodoWrite state to disk every N turns via hook
- Context compression: when context > 80% full, summarize completed steps and compress
- Structured session state: replace file-based context-save/restore with a typed state schema

### 4. "What would you change about the testing infrastructure?"
**Angles:**
- Deterministic test replay: record claude -p NDJSON outputs and replay without API calls for regression detection
- Per-test cost caps: touchfiles.ts should enforce `maxCostCents` before execution
- Security regression tests: dedicated E2E tests for injection resistance
- Non-Claude host CI: at least one Codex-host E2E test in gate tier

### 5. "What would you change about the multi-agent coordination model?"
**Angles:**
- Add timeout enforcement on Agent tool calls (currently model-discretion)
- Add coordinator retry logic: if subagent fails, retry once before escalating
- Structured subagent result format: require JSON output from subagent rather than free-form text
- Add progress visibility: coordinator should log subagent progress to user (currently opaque)
