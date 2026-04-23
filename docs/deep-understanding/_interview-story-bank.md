# _interview-story-bank.md

> Ready-to-tell interview stories grounded in gstack's architecture. Each story follows a consistent format: setup → architectural point → trade-off → risk → improvement path. Use these as scaffolds, replace "we" with "I" or "my team" as the engagement level warrants.

---

## System Design Stories

### S1. Persistent daemon for sub-second AI tool calls

**Setup**: An AI agent driving a browser paid 2-3 seconds per command for Chromium cold-start via `npx playwright`. A typical 20-command workflow burned 40 seconds on overhead alone, and stateful browsing (cookies, localStorage, tabs) was impossible across commands.

**Architectural point**: We built a Bun HTTP daemon that keeps Chromium warm. The CLI speaks HTTP with a bearer token; state persists in `~/.gstack/browse.json`. Startup is atomic-locked via a `wx` file-create flag; state writes use `.tmp`+rename; idle shutdown reclaims memory after 30 minutes. A second `Bun.serve` binds a separate tunnel port with a locked path and command allowlist — ngrok forwards only that port, so remote callers can't reach admin endpoints even with correct headers.

**Trade-off**: Process complexity in exchange for latency. The daemon must survive crashes, must handle concurrent CLI callers, must not leak memory, must stay isolated on shared machines. We accepted this because 20x latency improvement on typical workflows justified the operational surface.

**Risk**: Singleton Chromium per user means concurrent skill sessions share tab state — a real production issue we haven't yet solved with per-session contexts.

**Improvement path**: Per-Claude-session Playwright browser contexts. One Chromium, N isolated contexts, scoped by session ID derived from the client environment.

---

### S2. Prompt compile pipeline for multi-host skill distribution

**Setup**: We had 40+ AI workflow templates (skills) across 10 host environments (Claude, Codex, Factory, OpenCode, ...). Copy-paste of shared preamble sections was drifting. Each host needed different tool names, path conventions, and frontmatter fields.

**Architectural point**: We built a TypeScript-to-markdown compiler. `SKILL.md.tmpl` files contain `{{PLACEHOLDERS}}`; a static `Record<string, ResolverFn>` maps placeholders to typed functions; one regex pass expands everything; unresolved placeholders throw a build error. Per-host frontmatter transforms (allowlist/denylist modes, descriptionLimit, conditionalFields, field rename) let the same template generate valid skills for 10 targets. A CI check runs the compiler in dry-run mode and fails if any committed SKILL.md has drifted from its template.

**Trade-off**: Zero runtime flexibility in exchange for complete build-time determinism. A new contributor can't ship a custom skill without learning the resolver contract. We accepted this because the alternative (runtime interpretation) would have allowed drift to compound invisibly.

**Risk**: Resolver files grew to 50KB+ single files (`review.ts`, `design.ts`) mixing unrelated concerns. Navigation and targeted edits suffer.

**Improvement path**: Split resolvers by concern (`review-sql.ts`, `review-concurrency.ts`), export from a `review/index.ts` that preserves the `{{REVIEW_ARMY}}` interface.

---

### S3. Six-signal prompt injection defense with ensemble voting

**Setup**: An AI agent reading web content faces prompt injection from hostile pages. Single-ML-classifier approaches produced too many false positives (Stack Overflow pages contain "ignore previous instructions" as prose).

**Architectural point**: Six signals, in three categories:
- Content transforms (L1 datamarking, L2 hidden-element strip, L3 URL filter) run in-process, always, on every path.
- ML classifiers (L4 TestSavantAI ONNX, L4b Claude Haiku transcript) run in a separate subprocess because `@huggingface/transformers` can't `dlopen` ONNX from a compiled Bun binary.
- Canary (L5) injects a random string into page content; if it escapes, something exfiltrated it.
- `combineVerdict` (L6) requires 2-of-N ML classifiers at WARN threshold (0.60) before escalating to BLOCK. Canary leak always BLOCKs.

**Trade-off**: Architectural complexity (two processes, cross-process state) in exchange for lower false-positive rate. Stack Overflow stopped getting blocked; real exfiltration still triggers.

**Risk**: The module boundary — `security-classifier.ts` must never be imported from the compiled binary — is enforced only by a file-top comment. A developer who accidentally imports it gets a silent breakage on user machines.

**Improvement path**: A dependency-cruiser rule or a 20-line `bun test` that greps import statements for banned edges.

---

## Reliability Stories

### R1. Diff-based test selection for paid evals

**Setup**: E2E tests cost ~$4/run. A full suite per PR was unsustainable on engineering velocity.

**Architectural point**: Each test declares its source-file dependencies in a touchfiles table. `git diff --name-only` produces the changed set. A test runs if any of its touchfiles intersect the changed set. Global touchfiles (session-runner, eval-store, touchfiles itself) trigger every test. `EVALS_ALL=1` forces everything. A tier classification (gate vs periodic) filters further.

**Trade-off**: A test whose touchfile list is wrong can silently skip when it should run. We accepted this because the cost savings were dramatic (typical PR dropped to $0-2) and the failure mode (stale touchfile) surfaces on first real regression.

**Risk**: No per-PR budget cap. A malformed diff plus `EVALS_ALL=1` could burn $50+ before anyone notices.

**Improvement path**: `GSTACK_MAX_PR_COST_USD` env var that aggregates total spend across `bun test:evals` and aborts further runs if the threshold is crossed.

---

### R2. Atomic state file writes with concurrent-safe startup

**Setup**: A persistent daemon needs a state file (PID, port, auth token). Multiple CLI clients could start the daemon concurrently. Partial reads during writes would corrupt state.

**Architectural point**: Startup acquires an atomic `wx` file-create lock; any racing client blocks. State is written via `.tmp` + `fs.renameSync` — rename is atomic on POSIX, so readers either see the old full content or the new full content, never a partial write. State file is `0o600` so other users on the same machine can't read it.

**Trade-off**: Platform-dependent. On Windows, behavior subtly differs; we had to add a Node.js fallback server because Bun's Chromium pipe handling doesn't work the same way.

**Risk**: If `rename` ever fails mid-operation (disk full, permission error), the `.tmp` file survives and the original is intact — but no user surface exists for this case.

**Improvement path**: Explicit ENOSPC handling with user warning; state file validation on read.

---

### R3. Dual-listener port separation instead of header-based origin checking

**Setup**: We needed to expose a subset of the daemon's API to remote AI agents via ngrok, while keeping admin endpoints local-only.

**Architectural point**: Two `Bun.serve` calls bind separate ports. The local listener accepts all paths with bearer auth. The tunnel listener accepts only `/connect`, `/command`, `/sidebar-chat` (path allowlist enforced before route dispatch) and only 17 browser-driving commands (command allowlist enforced inside the handler). ngrok forwards only the tunnel port. Remote callers physically cannot reach the local-only surface.

**Trade-off**: Two listeners instead of one. Slightly more complex lifecycle. Worth it because header-based origin checks are spoofable by any upstream proxy, whereas port separation is uncheatable.

**Risk**: A future developer adding a new admin-only endpoint might forget to check the surface parameter, exposing it over the tunnel.

**Improvement path**: Structured route declaration that makes surface scope a required field.

---

## Trade-off Stories

### T1. Regex instead of classifier for model routing

**Setup**: Sidebar chat needed to route between Opus (thorough analysis, expensive) and Sonnet (action execution, cheap). Wrong routing doubles cost or halves quality.

**Architectural point**: 15 lines of regex. `ANALYSIS_WORDS` routes to Opus; `ACTION_PATTERNS` (short imperatives) routes to Sonnet; default Opus on ambiguity. A code comment states the reasoning: "sonnet vs opus on 'click @e24' is 5-10x in latency and cost, with zero quality difference."

**Trade-off**: Accuracy vs simplicity. An ML intent classifier would handle more edge cases; a regex covers 90%+ of our observed intent space at zero inference cost. The 15-line regex lost nothing measurable.

**Risk**: Intent drift. New sidebar usage patterns could degrade the regex's accuracy over time.

**Improvement path**: Instrument routing decisions; review monthly. If regex accuracy drops below a threshold, reconsider.

---

### T2. Files-and-grep instead of vector DB for learnings

**Setup**: Cross-session memory across skill invocations. "Did we hit this bug before?" retrieval.

**Architectural point**: Learnings are text files in `~/.gstack/learnings/`. Retrieval is `grep` or `gstack-learnings-search`. No embeddings. No vector database.

**Trade-off**: No semantic retrieval in exchange for zero infrastructure, complete debuggability, and privacy (learnings never leave the user's machine). We accepted this because our learnings volume is low (tens per project) and exact keyword retrieval works at this scale.

**Risk**: As learning count grows, exact-match retrieval degrades. Users would eventually report "I solved this once but can't find my note."

**Improvement path**: Local SQLite with FTS before reaching for embeddings. Still inspectable, still private, still no external infra.

---

### T3. Dynamic cost accounting instead of flat per-test budget

**Setup**: Paid LLM evals. Need to know what each run actually costs.

**Architectural point**: `pricing.ts` has per-model rate tables. Each test's cost is `(input + cached × 0.1 + output) × model_rate / 1_000_000`. The eval store records dollars per run for trend comparison.

**Trade-off**: A small amount of compute overhead + a pricing table to maintain, in exchange for real cost visibility. We decided seeing true spend was worth it.

**Risk**: No budget kill switch. Real costs are visible after the fact; accidents during live runs aren't prevented.

**Improvement path**: Aggregate spend across a single `bun test:evals` invocation; abort if over threshold.

---

## "What I Would Improve" Stories

### I1. Mechanical enforcement of module boundaries

**Setup**: Two load-bearing architectural invariants in gstack: `security-classifier.ts` must not be imported from `server.ts`; `token-registry.ts` must not be imported from `sse-session-cookie.ts`. Both are enforced by file-top comments only.

**Current state**: Documentation. If a developer violates one, users see a silent runtime failure on their machine (ONNX `dlopen` error on compiled binary startup).

**What I'd do**: Add a dependency-cruiser config with 5 lines of YAML. Alternatively, a 20-line `bun test` that reads every source file's import list and fails on banned edges. Costs one hour of work; closes a real silent-failure mode.

---

### I2. Checkpoint-resume for `/ship`

**Setup**: `/ship` is a 19-step workflow that can take 30+ minutes. If the daemon idle-times out (30 min no activity), or if Claude context gets compacted, the skill loses progress.

**What I'd do**: Each step writes a checkpoint to `.gstack/ship-state.json` — `{step: N, completed: [...], pending: [...]}`. On re-invocation, the skill reads the file and resumes from the last completed step. The preamble instructs Claude to check for this state file on entry.

**Why it matters**: The Boil-the-Lake philosophy says "do the complete thing" — but long workflows are exactly where interruption risk is highest. Resume turns a 30-minute gamble into a series of small checkpoints.

---

### I3. `temperature: 0` + structured output on the LLM judge

**Setup**: The eval system compares judge scores across runs to detect regression. Current judge implementation calls Anthropic without setting temperature (API default, ~1.0) and extracts JSON from a plain-text completion via regex.

**What I'd do**: Pass `temperature: 0` on the API call. Replace regex extraction with `tool_choice` and a JSON schema. Zero additional cost; reproducible scores.

**Impact**: Eval trends become meaningful. Today, comparing Tuesday's run to Wednesday's run is ambiguous because score variance could mask real changes.

---

## CTO / Executive Framing Stories

### E1. Effort compression at the workflow level

**Frame for a CTO**: gstack captures complete engineering workflows as opinionated skills. Our `/ship` doesn't just commit-and-push — it runs tests, bumps version, writes changelog, pushes, opens PR, runs a canary verification after deploy. 19 steps. Takes 30 minutes. Replaces ~2 engineer-hours of manual work.

**Architectural point**: The philosophy — "AI makes completeness cheap" — is baked into every workflow. Skills prefer long-form, complete execution over shortcut modes because the AI cost is marginal compared to the engineer's time savings.

**Risk**: Opinionation. Teams with different conventions may find skills inflexible. Migration cost is real.

**Improvement path**: Skills should expose per-team config surface (extension points for company-specific review rubrics, CHANGELOG formats, shipping processes).

---

### E2. Security posture as a competitive feature

**Frame for a CTO**: The security stack is architectural, not bolted on. Two-process split forced by tooling became a defensive advantage — ML classification doesn't slow the hot path. Ensemble + deterministic canary kills the false-positive class that makes one-off classifiers useless in production.

**Decision-level point**: Single-user install is defensible today. Fleet deployment needs three additional investments — signed skill releases, per-team budget dashboards, operational runbooks. Each is 2-5 days of work. Total: ~2 engineer-weeks to fleet-ready.

**Risk**: A supply-chain compromise of the gstack repo reaches every `/gstack-upgrade` on the next install. We don't sign releases today.

**Improvement path**: Signed skill manifest + verification in the upgrade skill. Investment scales with adoption.

---

### E3. Vendor exposure and portability

**Frame for a CTO**: Claude Code is the agent runtime. Anthropic SDK for judge and sidebar. HuggingFace for ML models. Supabase for telemetry. Bun for build. Nothing else is load-bearing.

**Decision-level point**: Claude Code is the real bet. Everything else is swappable. Supabase → any Postgres. HuggingFace → any ONNX. Bun → Node (Windows fallback already exists). Playwright → Puppeteer. We are not stuck on any non-Anthropic vendor.

**Risk**: If Claude Code fundamentally changes the agent contract (new tool protocol, new permission model), the skill system has to migrate. This is a bet, not an accident.

**Improvement path**: Every non-trivial host change is version-tagged; migration scripts run on upgrade. This is how we survived the last three Claude Code releases.

---

## Format Reminder

Every story above follows: **setup → architectural point → trade-off → risk → improvement path**. Use this format for any new story. The point is to demonstrate architectural thinking, not to recite features. Strong interviewers want to hear you:

1. Frame the problem clearly (setup).
2. Describe the non-obvious design choice (architectural point).
3. Acknowledge what you gave up (trade-off).
4. Name what could still go wrong (risk).
5. Know how you'd fix it (improvement path).

A story missing any of these five is weaker than it could be.
