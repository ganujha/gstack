# 09-teaching-note.md — gstack

---

## For a Strong Software Engineer New to Agentic Systems

gstack is a skill platform that supercharges Claude Code with specialized AI workflows. Here's how to think about it:

**The core idea**: Claude reads a markdown file (SKILL.md) that tells it exactly how to execute a workflow — like shipping code, reviewing PRs, or debugging bugs. These markdown files are generated from TypeScript templates, so they're maintainable and composable.

**The browser tool**: Most interesting tasks require a browser. Cold-starting Chromium for every command would be 2-3 seconds per step — too slow for a 20-step workflow. gstack solves this with a persistent HTTP server (`browse/src/server.ts`) that keeps Chromium warm. Your Claude session talks to this server with HTTP POST requests. This is the architectural key insight: **the tool is a long-running server, not a script**.

**How skills work**:
1. You run `./setup` — it builds a Chromium binary and links skill markdown files into `~/.claude/skills/`
2. In Claude Code, you type `/ship` — Claude reads `ship/SKILL.md` into its context
3. Claude executes the 19-step workflow, using Bash/Read/Edit/Write tools plus `$B` (browse commands)
4. The browse commands hit the HTTP server → Playwright → Chromium

**What to look at first**:
- `browse/src/commands.ts` — the 50 browser commands (your tool surface)
- `ship/SKILL.md` — a complete generated skill (what Claude reads)
- `browse/src/cli.ts` — how the CLI starts and manages the server
- `ETHOS.md` — the "Boil the Lake" philosophy (completeness is cheap with AI)

**Key concept — @refs**: When Claude asks the browser for a snapshot, it gets back an ARIA accessibility tree with `@e1`, `@e2` labels for interactive elements. Claude then says `$B click @e1` instead of figuring out a CSS selector. This is token-efficient and avoids selector brittleness.

**The security layer exists because web content is untrusted**: A web page can try to inject instructions like "ignore previous instructions." gstack defends against this with 6 layers including ML classifiers and a canary token system.

---

## For a Staff+ Engineer

gstack is a production AI engineering platform. Here's the architecture that matters:

**Prompt compilation pipeline**: Skills are not hand-written prompts. They're compiled from TypeScript resolvers into multi-host markdown files. The pipeline: `.tmpl` file with `{{PLACEHOLDER}}` markers → `gen-skill-docs.ts` → resolver functions generate sections → host-aware frontmatter transforms → SKILL.md output for each of 10 AI agent hosts. This is a prompt compiler. The design choice is sound: type safety at build time, DRY shared sections (preamble, review specs), CI freshness enforcement.

**Preamble tier system**: Skills declare a tier (1-4) in frontmatter. Each tier adds behavioral contracts: Tier 1 = core ops, Tier 2 = adds confusion protocol + context recovery, Tier 3 = adds repo-mode + search-before-building, Tier 4 = adds specialized sections. Adding a new behavioral instruction to all Tier-2+ skills requires modifying one generator file. This is the right abstraction for a 40-skill system.

**Browse daemon design decisions**:
- Bun.serve (not Express/Fastify): single binary, no node_modules
- Random port (10000-60000): survives multiple Conductor workspaces without conflicts
- `wx` flag lock file: O(1) atomic startup serialization, POSIX-guaranteed
- HTTP health check (not process table): more reliable liveness signal
- Physical port separation (not header-based): unforgeable security boundary
- Binary version hash in state file: automatic stale-server detection and restart

**Eval infrastructure**: Three tiers with cost engineering:
- Free (<2s): static validation, catches most template errors before any API calls
- Gate (~$4): LLM-judge + targeted E2E, runs on every PR but only for changed skills (diff-based via touchfiles.ts)
- Periodic (~$3.85): full E2E suite, weekly cron

The touchfiles.ts approach is smart: each test declares its file dependencies as glob patterns, and git diff determines which tests run. The alternative (run everything) costs ~$400/month at typical PR volume.

**Reliability gaps to note**: No atomic writes for `browse.json` state file (partial reads possible during concurrent startup), no worktree isolation for concurrent Claude Code sessions writing CHANGELOG.md/TODOS.md, Windows CI coverage is minimal (only Linux in CI despite Node.js fallback existing).

**The security architecture** is among the most sophisticated in any open-source AI agent project. The ensemble rule (2-of-3 ML classifiers at 0.60 threshold → BLOCK) was specifically designed to reduce Stack Overflow false positives where instruction-style text triggers classifiers. The canary tripwire provides a deterministic fallback that's immune to classifier error. The physical port separation (not header inspection) for the tunnel is the correct engineering choice — headers can be forged, port binding cannot.

**What this project gets right at scale**: The "Boil the Lake" philosophy is operationalized in code. Skills don't cut corners — the `/ship` workflow does all 19 steps. The eval system proves completion, not just start. This is the right posture for an AI system that claims to compress weeks of work into minutes.

---

## For a CTO Evaluating the Architecture

**The strategic thesis**: gstack bets that the highest-leverage place to invest in AI productivity is at the **workflow level**, not the model level. A better model makes each step faster; better workflows eliminate entire categories of steps. The `/ship` skill makes the claim concrete: 19-step shipping workflow, fully automated, with human control at exactly the decision points that matter (merge conflicts, failing tests, plan items not done). The rest runs autonomously.

**What makes this defensible**: The skill system has 18 months of opinionated workflow content baked in — review checklists, security patterns, CHANGELOG formats, version bump strategies. This isn't raw capability; it's curated judgment. The `review.ts` resolver is 54K of opinionated review content. That's not replicable by prompting a base model; it's the result of Garry Tan's engineering philosophy encoded in TypeScript.

**Key architectural bets**:
1. **Persistent browser daemon over serverless**: Right for AI agents doing multi-step workflows. Wrong for single-query use cases. gstack's use case is multi-step (20+ commands per session), so the persistent model wins.
2. **Prompt compilation over runtime assembly**: Smart for a system with 40+ skills and 10 hosts. The build step adds friction for contributors but eliminates runtime complexity and enables type safety.
3. **File-system memory over vector store**: Learnings and state in `~/.gstack/` files. Simple, debuggable, no external dependency. The trade-off is no semantic search over learnings — but for a single-user tool, JSONL + grep is often good enough.
4. **6-layer prompt injection defense**: This is production-grade. Web pages are an adversarial content surface. The ensemble ML + deterministic canary architecture would hold up to a security audit. Most AI browser tools have zero injection defense.

**What I'd want to validate before adoption**:
- Concurrent skill execution story: two windows running `/ship` at once will collide on CHANGELOG.md/VERSION (no file locking visible)
- Windows support: Node.js fallback exists but CI coverage is Linux-only
- Non-Claude host fidelity: Codex and Factory skill generation exists but test coverage is uncertain
- Cost at scale: ~$8/PR for gate + E2E is fine for a team of 5; at 50 PRs/week it's $400/month in API costs

**Production readiness verdict**: Ready for a technical founder or small team wanting to operate at 10x throughput. Not ready for enterprise deployment without: centralized auth, multi-user state isolation (currently assumes single user per machine), Windows first-class support, and audit logging beyond the local attack log.

**The compelling use case**: A solo technical founder shipping at team speed. The `/ship` workflow + `/review` + `/investigate` + `/canary` covers the full deploy lifecycle. The browser daemon handles QA. The retro skill handles weekly reflection. This is a complete engineering workflow system, not a collection of tools.
