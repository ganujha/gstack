# 07-open-questions.md — gstack

## Unclear Parts of the Codebase

### 1. Supabase Backend Scope
**Question**: What does the Supabase backend (`supabase/functions/`, `supabase/migrations/`) actually store and serve?
**Inferred**: Likely telemetry data synced by `bin/gstack-telemetry-sync`, possibly analytics for review patterns or skill usage.
**Uncertain**: Whether Supabase is used in production for any user-facing feature or purely for internal analytics.
**Verify**: Read `supabase/functions/` directory and `bin/gstack-telemetry-sync` source.

### 2. sidebar-agent.ts Location and Role
**Question**: CLAUDE.md references `sidebar-agent.ts` as a separate process started in headed mode. Where does it live?
**Inferred**: Likely in `browse/src/sidebar-agent.ts` or as a separate entry point.
**Uncertain**: Whether sidebar-agent is a distinct binary or just a module. Whether it uses the Agent SDK or `claude -p`.
**Verify**: `find /home/user/gstack -name "sidebar-agent*"`

### 3. GBrain Integration
**Question**: What is GBrain? Is it an external service, a local model, or an internal gstack component?
**Evidence**: `hosts/gbrain.ts`, `scripts/resolvers/gbrain.ts` exist. `learningsMode` is set to `'full'` (vs. `'basic'` for others, inferred).
**Uncertain**: Whether GBrain requires external auth, whether it's a product of Garry Tan's portfolio, whether it's in production use.
**Verify**: Read `hosts/gbrain.ts` full contents.

### 4. Security Classifier Offline Fallback
**Question**: What happens on first run before the TestSavantAI model (112MB) finishes downloading? Does it block or allow?
**Inferred**: Probably degrades gracefully (skips L4 and runs without ML), but this is not confirmed from available evidence.
**Verify**: Read `browse/src/security-classifier.ts` initialization code; check for a `ready` flag or lazy-load pattern.

### 5. Learnings System Format
**Question**: What is the exact format of `~/.gstack/learnings/` entries? How are they retrieved and injected?
**Inferred**: The `scripts/resolvers/learnings.ts` resolver generates a shell command to search the learnings store. Likely JSONL or a structured directory.
**Verify**: Read `scripts/resolvers/learnings.ts` full contents + `bin/gstack-learnings-log` source.

### 6. Full Extent of Snapshot @ref Invalidation
**Question**: Are @refs explicitly invalidated when a navigation occurs? Or does the agent rely entirely on prompt instructions to re-snapshot?
**Inferred**: The locator map is stored server-side (BrowserManager). On navigation, Playwright's page object changes → old locators would fail. But whether there's explicit cache invalidation or silent failure is unclear.
**Verify**: Read `browse/src/browser-manager.ts` full contents; check for `page.on('framenavigated')` handlers.

### 7. Conductor Integration Details
**Question**: What is `conductor.json` at the repo root? Is Conductor an external AI product? How does gstack integrate?
**Inferred**: Conductor is likely a multi-agent workspace runner. `conductor.json` probably declares skills or commands Conductor can invoke.
**Verify**: Read `conductor.json` full contents; search for Conductor documentation.

### 8. Whether security-classifier.ts Is Present
**Question**: CLAUDE.md says `security-classifier.ts` CANNOT be imported from the compiled browse binary. Is it a separate runtime or does it run in a separate process?
**Inferred**: It likely runs in `sidebar-agent.ts` (which is not compiled into the browse binary) and communicates results via `session-state.json`.
**Verify**: `find /home/user/gstack -name "security-classifier.ts"`

### 9. Per-Command Security Decision Timing
**Question**: Does the security check run before or after page content is returned to the agent? I.e., is L4 blocking (synchronous) or advisory (async)?
**Inferred**: Blocking, because the server must return either content or a block response — it can't return content and later revoke it.
**Verify**: Read `browse/src/server.ts` handler for `text`/`snapshot` commands and trace the security call order.

### 10. gen-skill-docs.ts Resolver Registry Format
**Question**: Is `scripts/resolvers/index.ts` a static map or a dynamic discovery mechanism?
**Inferred**: Likely a static `Record<string, ResolverFn>` since the file is ~4.4K (small for dynamic discovery).
**Verify**: Read `scripts/resolvers/index.ts` full contents.

---

## Ambiguous Architecture Areas

### A. Local vs. Remote Eval Storage
CLAUDE.md mentions `~/.gstack-dev/evals/` (legacy) and `~/.gstack/projects/$SLUG/evals/` (current). It's unclear whether both are active or if the legacy path is fully deprecated.

### B. Headed Mode vs. Headless Mode Boundary
CLAUDE.md describes headed mode (Chrome Extension) and headless mode (CLI only). The transition between them (the `connect` command) is described, but the exact capability differences (which commands are only available in headed mode) are not fully documented from available evidence.

### C. Codex Skill Filesystem Boundary Enforcement
The Codex skill instructs Codex not to read `~/.claude/` directories — but this is a prompt-level instruction, not an OS permission. The enforcement model is entirely trust-based. No sandbox or chroot is in place.

### D. Multi-Workspace Chromium Instances
CLAUDE.md mentions "10 Conductor workspaces, zero conflicts" via random port allocation. But the mechanism for managing 10 independent `~/.gstack/browse.json` state files (one per workspace) is unclear. Does each workspace get its own state directory?

---

## Assumptions Made During Analysis

1. **`sidebar-agent.ts` is a separate module/process** — assumed from CLAUDE.md's description of "sidebar-agent subprocess" but not directly observed in directory listing.
2. **Security classifier runs in sidebar-agent, not browse binary** — assumed from CLAUDE.md constraint (ONNX can't dlopen from compiled Bun binary).
3. **GBrain is an external AI service** — assumed from it having a separate host config; could be internal.
4. **Learnings are JSONL or structured directory** — assumed from `bin/gstack-learnings-log` naming convention.
5. **Supabase is for analytics/telemetry** — assumed from co-location with `bin/gstack-telemetry-sync`; could serve other purposes.
6. **@refs expire on navigation** — assumed from Playwright semantics; not directly verified from snapshot.ts code.
7. **LLM judge uses temperature 0** — assumed from reproducibility requirements; not confirmed in llm-judge.ts.
8. **Security classifier is lazy-loaded** — assumed from 112MB model size and "first run only" language in CLAUDE.md; may block startup.

---

## What Should Be Validated by Running the System

1. **Start the browse daemon** and observe `~/.gstack/browse.json` structure
2. **Run `$B snapshot`** and observe @ref format, ARIA tree depth, flag effects
3. **Run `bun test`** and observe which tests pass/fail and in what order
4. **Run `bun run eval:select`** on a clean branch and verify no tests are selected
5. **Modify a resolver file** and run `bun run gen:skill-docs --dry-run` to observe placeholder expansion
6. **Run `./setup --dry-run`** (if supported) or trace the setup script on a fresh install
7. **Check `~/.gstack/` directory structure** to verify actual state file layout
8. **Trigger a security block** by loading a known injection payload and inspecting `attempts.jsonl`
9. **Run `bun run skill:check`** to see the health dashboard format
10. **Run one E2E test** in foreground (`bun test test/skill-e2e-ship.test.ts`) and observe NDJSON streaming

---

## What Is Inferred vs. Directly Confirmed

| Topic | Status | Basis |
|-------|--------|-------|
| Browse daemon is Bun HTTP server | CONFIRMED | browse/src/server.ts code |
| @ref invalidation on navigation | INFERRED | Playwright semantics |
| Supabase = analytics backend | INFERRED | Co-location with telemetry binary |
| GBrain is external service | INFERRED | Separate host config |
| sidebar-agent is separate process | CONFIRMED (from CLAUDE.md) | Documentation |
| Security classifier in sidebar-agent only | CONFIRMED (from CLAUDE.md) | Documentation |
| Learnings format is JSONL | INFERRED | Naming convention |
| LLM judge uses temperature 0 | INFERRED | Reproducibility requirement |
| Conductor = external product | INFERRED | conductor.json at root |
| @refs are server-side locator map | CONFIRMED | snapshot.ts description |
| Eval store uses project-scoped dirs | CONFIRMED | eval-store.ts reference + CLAUDE.md |
| Token ceiling is advisory, not hard | CONFIRMED | CLAUDE.md explicit statement |
