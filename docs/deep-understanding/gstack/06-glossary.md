# 06-glossary.md — gstack

## Repo-Specific Concepts

**$B** — Shell alias or invocation shorthand for the browse binary (`browse/dist/browse`). Used in skill templates and documentation. `$B goto https://example.com` runs a browse command against the persistent daemon.

**SKILL.md** — A markdown file read by an AI agent (Claude, Codex, etc.) as a system prompt or context document. Contains step-by-step workflow instructions. Generated from a `.tmpl` file.

**SKILL.md.tmpl** — The hand-authored source template for a skill. Contains `{{PLACEHOLDER}}` markers expanded at build time by `gen-skill-docs.ts`. Never edited after generation.

**Preamble** — The shared instruction block prepended to every skill's prompt. Controls agent behavior: context recovery, confusion protocols, writing style, telemetry, upgrade checks. Assembled from 22 generators based on preamble tier.

**Preamble tier** — Integer (1-4) declared in a skill's `.tmpl` frontmatter. Higher tier = more preamble sections = more self-regulation behavior. Tier 1 is minimal (browse); Tier 4 is fully orchestrated (ship, review).

**Resolver** — A TypeScript function in `scripts/resolvers/*.ts` that generates a section of skill prompt content. Each `{{PLACEHOLDER}}` in a `.tmpl` file maps to one resolver function.

**Host** — An AI agent platform that can consume gstack skills (Claude, Codex, Factory, OpenCode, Cursor, OpenClaw, Hermes, GBrain, Kiro, Slate). Each host has a `HostConfig` in `hosts/`.

**HostConfig** — TypeScript interface defining how skills are transformed for a specific AI agent host: frontmatter transforms, path rewrites, tool renames, install strategy.

**@ref** — A stable DOM element identifier assigned by the snapshot system (e.g., `@e1`, `@e42`). The agent uses refs in write commands (`$B click @e1`) rather than CSS selectors or XPath. Refs are valid only until the next page navigation.

**Browse daemon** — The persistent Chromium HTTP server (`browse/src/server.ts`). Starts on first `$B` command, listens on a random port (10000-60000), stays alive 30 minutes after last command.

**Dual-listener topology** — Security architecture where the browse server binds two separate TCP ports: a local port (full command surface) and a tunnel port (restricted allowlist). ngrok forwards only the tunnel port.

**Snapshot** — ARIA accessibility tree of the current page, captured as structured text with @ref annotations. Provides the agent's "view" of the browser without screenshots (text-efficient).

**Skill** — A complete AI workflow delivered as a SKILL.md file. Examples: `/ship` (full shipping workflow), `/review` (code review), `/investigate` (root cause debugging). Invoked via slash command in Claude Code.

**Boil the Lake** — Core philosophy from ETHOS.md: "AI makes completeness cheap; always do the complete thing rather than a shortcut." Referenced in the preamble to prevent scope-cutting.

**Touchfile** — An entry in `test/helpers/touchfiles.ts` that declares which source files a given E2E test depends on. Changes to those files trigger the test; other tests are skipped.

**Gate tier** — E2E tests that must pass before a PR can merge (safety guardrails, deterministic functional tests). Runs in CI via `bun run test:gate`.

**Periodic tier** — E2E tests that run weekly or manually (quality benchmarks, non-deterministic, external services). Not required for merge. `bun run test:periodic`.

**LLM judge** — A Claude Sonnet call that scores a generated SKILL.md section on clarity (≥4), completeness (≥3), and actionability (≥4) on a 1-5 scale. Output persists in eval store.

**Eval store** — Persistent JSON store at `~/.gstack/projects/$SLUG/evals/` (managed by `test/helpers/eval-store.ts`). Schema-versioned. Enables trend comparison across PRs.

**Session runner** — `test/helpers/session-runner.ts`. Spawns `claude -p` as a subprocess, feeds prompt via stdin, streams NDJSON output, returns `SkillTestResult`.

**AskUserQuestion** — A Claude Code tool that pauses skill execution and presents a structured question to the user. Used when a decision requires human judgment. Skills declare when they must vs. must not stop.

**Canary** — L5 of the security stack. A secret token injected into page content before the agent sees it; if the token appears in the agent's outbound actions, the session is blocked. Deterministic, unlike ML layers.

**Datamarking** — L1 of the security stack. Zero-width spaces + session marker injected between sentences. Intended to detect exfiltration by watermarking text content.

**combineVerdict** — Function in `browse/src/security.ts` that aggregates signals from all security layers into a final verdict (`safe`, `warn`, `block`). Implements the ensemble rule: 2-of-3 ML at WARN → BLOCK.

**GSTACK_SECURITY_OFF** — Emergency environment variable that disables ML security classifiers. Canary injection still fires.

**pair-agent** — A browse command that generates a setup key + instruction block for a remote AI agent. Enables a remote Claude or other agent to drive the local browser via the tunnel port.

**Worktree** — A git feature creating an isolated checkout of the repo at a specific commit. `lib/worktree.ts` manages worktrees for E2E test isolation (prevents one test from contaminating another's file state).

**Slop-scan** — Tool (`npx slop-scan`) that detects patterns where AI-generated code is genuinely worse quality. Used in CI; not trying to hide AI origin, just catch empty catches, redundant awaits, etc.

**gstack-slug** — CLI utility in `bin/gstack-slug`. Derives a project identifier from the current repo for namespacing eval results and other per-project state.

**skill_prefix** — Config option in `~/.gstack/config.yaml`. When `true`, skills are installed as `gstack-ship`, `gstack-review`, etc. instead of `ship`, `review`.

**Conductor** — External product (separate from gstack) that manages multiple workspace environments. `conductor.json` at repo root configures gstack for Conductor. Each workspace gets an independent Chromium process.

**GBrain** — A host type in `hosts/gbrain.ts`. Likely an agent runtime that can consume gstack skills. Integration details partially unclear.

---

## Project Vocabulary

**fix-first review** — Review approach where obvious issues are auto-fixed before asking the user questions. Reduces interruptions for mechanical improvements.

**review army** — Multi-specialist review pattern: CEO, Design, Eng, DX reviewers each examine different aspects of a PR. Coordinated by `scripts/resolvers/review-army.ts`.

**planted-bug fixture** — E2E test fixture with a deliberately injected bug. The `outcomeJudge()` function scores whether Claude correctly identified and fixed it.

**explain_level** — User config option (`terse` or default). `terse` skips explanatory prose in skill output.

**team mode** — Install option (`./setup --team`). Enables `auto_upgrade=true`, `team_mode=true`, SessionStart hook for repo bootstrap.

**link strategy** — How skills are installed into the AI agent's skill directory. `real-dir-symlink`: create a real directory, symlink only the SKILL.md inside. `symlink-generated`: symlink the entire generated directory.

**voice trigger** — Natural language phrases in a skill's frontmatter that Claude Code uses to auto-route to the skill. Processed by gen-skill-docs and folded into the skill description.

---

## Execution Lifecycle Terms

**health check** — HTTP GET `/health` on the browse daemon. Returns 200 if running. The only authoritative signal for server liveness (preferred over process table checks).

**server lock** — File opened with `wx` flag (create-only, atomic). Prevents two CLI invocations from starting two servers concurrently (TOCTOU prevention).

**binary version hash** — Hash of the compiled browse binary stored in server state. If CLI binary differs from running server's stored hash, server is restarted automatically.

**idle timeout** — 30 minutes of inactivity triggers browse daemon shutdown. Chromium profile is cleaned up.

**token** — Random string stored in `~/.gstack/browse.json`. Sent as Bearer auth on every HTTP command. Rotated on server restart.

**scoped token** — Restricted token issued to paired agents. Grants access only to the tunnel-surface allowlist (17 commands), not the full command surface.

**tunnel** — The restricted HTTP listener forwarded via ngrok. Only `/connect`, `/command` (with scoped token + 17-command allowlist), and `/sidebar-chat` are reachable.

---

## Memory / Tooling Terms

**in-context scratchpad** — TodoWrite tool state maintained within a Claude Code session's context window. Used by skills to track task progress. Lost on session exit.

**context recovery** — Preamble instruction telling Claude to re-read its TodoWrite state if context window was pruned during a long workflow.

**continuous checkpoint** — Preamble instruction to periodically write progress to disk during long workflows.

**learnings** — Cross-session knowledge accumulated by skills and stored at `~/.gstack/learnings/`. Used by the `scripts/resolvers/learnings.ts` resolver to inform future skill runs.

**eval baseline** — Previous eval run stored in eval-store. New runs are compared against baseline to detect regressions.

**NDJSON** — Newline-delimited JSON. Format used by `claude -p --output-format stream-json --verbose` output. `session-runner.ts` parses this stream into structured `SkillTestResult`.

**cost estimate** — Computed from NDJSON output by `session-runner.ts`. Tracks token cost per E2E test run. Persists in eval store for trend analysis.
