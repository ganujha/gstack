# 17-best-reading-order-v2.md — gstack

> Revised after the second pass. The first-pass walkthrough was structured around "most important files by size." The actual learning curve is better served by reading in order of **architectural clarity** — start with the compile step, then the runtime, then the orchestration layer.

---

## What the First Pass Overemphasized

- **`ship/SKILL.md.tmpl` as a starting point**. It's 42K tokens, it's the product's flagship, but it's downstream of the architecture. Reading it first is like starting with a compiled binary.
- **Resolver file sizes (`review.ts` 54K, `design.ts` 58K)**. These are tall, not deep. Their content is worth skimming, not studying. The registry shape (`scripts/resolvers/index.ts`) matters more than any single resolver file.
- **CHANGELOG.md (321K)**. Useful for flavor, not for understanding.

## What the First Pass Underemphasized

- **`scripts/gen-skill-docs.ts` placeholder loop (lines 430-448)**. Thirteen lines. Describes the entire compile model.
- **`browse/src/security.ts` module-boundary comment (lines 1-20)**. Explains why the system has two processes.
- **`test/helpers/pricing.ts`**. Tiny file, but it's the actual cost model. Understanding dynamic cost accounting changes how you read the eval infrastructure.
- **`hosts/claude.ts` vs `hosts/codex.ts` side-by-side diff**. The delta is where the multi-host architecture lives.
- **`browse/src/server.ts:190-211` (sidebar model routing)**. Tiny function that shows the cost-routing philosophy.

---

## Improved Reading Order (45-minute path)

### Phase 1 — Understand the compile step (8 minutes)

1. `scripts/gen-skill-docs.ts` — focus on `resolveTemplate` (~line 420-450) and `transformFrontmatter` (~line 200-300). Understand: one-pass, fail-fast, per-host frontmatter.
2. `scripts/resolvers/index.ts` — see the static `Record<string, ResolverFn>`. Confirm: placeholders are keys, resolvers are values, output is markdown.
3. `scripts/resolvers/preamble.ts` (lines 71-103) — see the tier composition.
4. `hosts/claude.ts` + `hosts/codex.ts` — read both; note what differs (mode: denylist vs allowlist, pathRewrites, descriptionLimit).

**You now understand**: how a template becomes a skill.

### Phase 2 — Understand the runtime (12 minutes)

5. `browse/src/cli.ts` — focus on `ensureServer`, `acquireServerLock`, `startServer`, `runCommand`. About 200 lines of the total.
6. `browse/src/server.ts` — read the two `Bun.serve` calls (lines 2712, 2785), the dual-listener `makeFetchHandler` (line 1564), the tunnel allowlists (lines 95-113), the idle shutdown (lines 899-909). Skip the 2,000+ lines of handler implementations.
7. `browse/src/security.ts` — read the module-level comment (lines 1-20), the canary logic (lines 180-226), the `combineVerdict` (lines 96-178). Don't read the rest.

**You now understand**: how `$B` commands reach Chromium, how the security ensemble gates untrusted content.

### Phase 3 — Understand the agent loop (8 minutes)

8. `ship/SKILL.md.tmpl` — read ONLY the first screen and the 19 step headers. Do not read the full body. You're learning the shape, not the content.
9. `autoplan/SKILL.md.tmpl` — same approach; learn the sequential phase pattern.
10. `scripts/resolvers/preamble/generate-confusion-protocol.ts` — short. See what goes into the tier-2 behavioral contract.

**You now understand**: how skills orchestrate the agent.

### Phase 4 — Understand the test infrastructure (8 minutes)

11. `test/helpers/touchfiles.ts` — see the `E2E_TOUCHFILES` record and `GLOBAL_TOUCHFILES` array.
12. `test/helpers/session-runner.ts` — see the `claude -p` spawn and NDJSON parse.
13. `test/helpers/llm-judge.ts` — see the judge call; notice the **lack** of temperature setting and structured output.
14. `test/helpers/pricing.ts` — see the per-model rate table and compute formula.

**You now understand**: how tests cost money, when tests run, how results persist.

### Phase 5 — Understand the extension (6 minutes)

15. `browse/src/sidebar-agent.ts` — top comment + the L4 + L4b calls around line 496.
16. `browse/src/security-classifier.ts` — top comment only. Understand *why* it can't be imported from the compiled binary.
17. `extension/manifest.json` + brief skim of `sidepanel.js`. Confirm: this is a Chrome extension consuming the daemon's HTTP API.

**You now understand**: the second process and what it exists for.

### Phase 6 — Calibrate on what NOT to go deep on (3 minutes)

18. Glance at `scripts/resolvers/review.ts` and `design.ts` — confirm they are long-form prose generators. Decide NOT to read in full.
19. Glance at `extension/sidepanel.js` — confirm it's 74K of vanilla JS. Decide NOT to read in full unless extension development is your focus.
20. Open `CHANGELOG.md` — read only the top entry. Confirm its release-summary format. Move on.

---

## Best Files For Understanding Each Concern

| Concern | Best file | Why |
|---------|-----------|-----|
| **Orchestration** | `ship/SKILL.md.tmpl` (first screen only) | All workflow decisions live in prompts |
| **Tool integration** | `browse/src/commands.ts` + `browse/src/write-commands.ts:click` | The tool surface that skills actually call |
| **Memory / state** | `browse/src/cli.ts readState/writeState` + `~/.gstack/projects/*/evals/*.json` schema at `test/helpers/eval-store.ts:92-108` | Where state lives and how it's shaped |
| **Runtime / lifecycle** | `browse/src/cli.ts:ensureServer` + `browse/src/server.ts:idle-shutdown` | Startup + steady state + teardown |
| **Config / distribution** | `hosts/claude.ts` + `hosts/codex.ts` diff | How one template becomes ten skills |
| **Reliability** | `browse/src/error-handling.ts` + `lib/worktree.ts:dedup+harvest` | Error envelope + test isolation |
| **Security architecture** | `browse/src/security.ts:1-20` (comment) + `browse/src/security-classifier.ts:2-10` (comment) | The two-process invariant explained by the authors |
| **Cost model** | `test/helpers/pricing.ts` (the whole file) | Tiny, precise, load-bearing |
| **Sidebar model routing** | `browse/src/server.ts:190-211` | The cleanest cost-routing example in the repo |
| **Dependency graph of skills** | `scripts/resolvers/preamble.ts` (tier composition) | One file, everything downstream |

## Best Files To Ignore Until Later

| File | Why ignore |
|------|-----------|
| `scripts/resolvers/review.ts`, `design.ts`, `testing.ts` | Long prose generators; content changes weekly; shape doesn't |
| `ship/SKILL.md.tmpl` full body | 42K tokens; shape (not content) is what you need to understand first |
| `CHANGELOG.md` | Reference material; consult for history, don't read front-to-back |
| `extension/sidepanel.js` | Only read if working on the sidebar |
| `bin/gstack-*` individual utilities | Each is standalone; skim names in `bin/`, read the one you need |
| `supabase/migrations/` | Only relevant when touching telemetry schema |
| `docs/designs/` | Historical design documents; helpful after you know the code, not before |

---

## Files To Read Early Only If You Care About Specific Topics

### Agent / skill system internals
- `scripts/resolvers/preamble/generate-context-recovery.ts`
- `scripts/resolvers/preamble/generate-continuous-checkpoint.ts`
- `scripts/resolvers/preamble/generate-writing-style.ts` (note: reads `jargon-list.json` at compile time)

### Security architecture
- `browse/src/content-security.ts` (L1-L3 — datamarking, hidden element strip, URL/content filter)
- `browse/src/tunnel-denial-log.ts` (attack log rotation logic)
- `browse/src/sse-session-cookie.ts` (cookie-based SSE auth)

### Test infrastructure deep dive
- `lib/worktree.ts` (isolation model for tests)
- `test/helpers/eval-store.ts` (full comparison logic around line 220-318)
- `test/helpers/skill-parser.ts` (how skill templates are parsed for validation)

### Build pipeline deep dive
- `setup` (bash; 1,012 lines; understand the install story)
- `browse/scripts/build-node-server.sh` (Windows fallback build)
- `gstack-upgrade/migrations/*.sh` (version migrations)

---

## Anti-Patterns To Avoid While Reading

1. **Don't start with CHANGELOG**. It's a product changelog, not an engineering guide. Reading it first gives you marketing context, not architecture.
2. **Don't read SKILL.md files as code**. They're prompts. Skim the shape; understand the composition; don't try to follow every step in detail until you understand how the compile step generated them.
3. **Don't try to fully understand `server.ts` in one sitting**. It's 2,835 lines and covers HTTP, Playwright, security, tabs, tunneling, and sidebar chat. Read the route dispatcher (makeFetchHandler) and the two Bun.serve calls, then move on.
4. **Don't ignore `ETHOS.md`**. The "Boil the Lake" and "Search Before Building" sections explain why the repo makes the choices it does.
5. **Don't treat resolver files as reading material**. They're generators — read them when you need to change them.

---

## After the 45-Minute Path: Where To Go Next

- **If you are evaluating gstack for your team**: read `14-architecture-critique.md` + `16-production-readiness-review.md`.
- **If you are contributing a new skill**: read `scripts/gen-skill-docs.ts` in full, then the existing `codex/SKILL.md.tmpl` as a reference example.
- **If you are contributing a new host**: read every file in `hosts/`, then `scripts/host-config.ts` for the interface.
- **If you are contributing to security**: read `browse/src/security.ts` + `content-security.ts` + `security-classifier.ts` in full. Read the Pre-Impl Gate 1 outcome doc referenced in CLAUDE.md.
- **If you are debugging a production issue**: read `13-runtime-behavior-deep-dive.md` first, then the specific handler for the failing command.
