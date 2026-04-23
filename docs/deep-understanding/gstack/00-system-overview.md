# 00-system-overview.md — gstack

## BLUF (Bottom Line Up Front)

gstack is an AI engineering productivity platform built by Garry Tan (CEO of Y Combinator) that extends Claude Code with 40+ specialized "skills" (markdown prompt templates), a persistent headless browser daemon (sub-100ms latency), a multi-host skill distribution system (Claude, Codex, Factory, OpenCode, and more), and a layered security architecture for browser-based AI agents. It compresses weeks of engineering workflow into minutes by embedding complete, opinionated workflows for shipping, reviewing, investigating, and designing software — installable as a single `./setup` command.

## Problem Being Solved

AI agents interacting with browsers and codebases face three compounding problems:

1. **Latency**: Cold-starting Chromium per command costs 2-3 seconds; 20 commands = 40+ seconds of pure overhead.
2. **Context loss**: Each new AI session forgets what the agent learned in prior sessions — about the project, the user, and accumulated patterns.
3. **Workflow incompleteness**: Generic AI assistants give point-in-time answers but lack the opinionated, end-to-end workflows needed to actually ship: review → test → bump version → write changelog → merge → deploy → canary check.

gstack solves all three:
- **Persistent Chromium daemon** keeps browser warm across commands
- **Skill system** bakes complete workflow opinions into Claude's context
- **Learnings + state** accumulate cross-session knowledge about the project and user

## Who the User/Operator Is

- **Primary user**: Garry Tan and YC founders / portfolio companies
- **Secondary users**: Technical founders, senior engineers, and AI-augmented solo developers who want to operate at "team" throughput (3x-100x effort compression per ETHOS.md)
- **Operator context**: The person has Claude Code installed and is working on a software project — they invoke skills via slash commands (`/ship`, `/review`, `/investigate`)

## Major Subsystems

| Subsystem | Directory | Role |
|-----------|-----------|------|
| **Browse daemon** | `browse/` | Persistent Chromium HTTP server; all browser automation |
| **Skill system** | `<skill>/SKILL.md.tmpl` + `scripts/resolvers/` | Prompt templates for 40+ AI workflows |
| **Host system** | `hosts/` + `scripts/gen-skill-docs.ts` | Multi-AI-agent skill distribution |
| **Security layer** | `browse/src/security.ts` + `content-security.ts` | Prompt injection defense (6 layers) |
| **Test infrastructure** | `test/` + `test/helpers/` | LLM-judge evals, E2E tests, skill validation |
| **State management** | `~/.gstack/`, `browse/src/cli.ts` | Server PID/port, user config, eval history |
| **Chrome extension** | `extension/` | Headed browser with sidebar chat UI |
| **CLI utilities** | `bin/` | 38 per-feature command-line tools |
| **Eval system** | `test/helpers/eval-store.ts`, `touchfiles.ts` | Diff-based test selection + result persistence |

## External Dependencies / Integrations

| Dependency | Version | Purpose |
|------------|---------|---------|
| Playwright | 1.58.2 | Chromium browser automation |
| Puppeteer-core | 24.40.0 | Supplemental browser control |
| Anthropic SDK | 0.78.0 | LLM-as-judge evals, sidebar model calls |
| @huggingface/transformers | v4 | ML prompt injection classifier (ONNX) |
| ngrok (optional) | runtime | Tunnel for pair-agent remote access |
| Supabase | runtime | Backend (analytics/telemetry, inferred) |
| GPT Image API | runtime | Design skill image generation |
| Codex CLI | runtime | Second-opinion review via `/codex` |
| GitHub Actions | CI | 6 workflows (evals, skill-docs, CI image) |

## What Makes This Project Interesting from an Agentic-Systems Perspective

1. **Skill-as-prompt pattern at scale**: gstack operationalizes 40+ complete workflows as markdown prompt files, compiled from typed TypeScript template resolvers — essentially a "prompt compiler" architecture.

2. **Persistent daemon for sub-second tool use**: Rather than stateless HTTP calls, the browse daemon maintains a warm Chromium process. This is a key architectural insight: the agent's tool is a long-running server, not a per-invocation script.

3. **Multi-layer prompt injection defense**: The security architecture has 6 distinct layers (datamarking, DOM stripping, ARIA regex, ML content classifier, ML transcript classifier, canary injection) — among the most comprehensive injection defenses in any open-source AI agent project.

4. **Preamble tier system**: Skills compose behavior by tier (1-4), each tier adding more self-regulation instructions (context recovery, confusion protocols, checkpoint behavior). This is a principled approach to controlling agent autonomy.

5. **Effort compression philosophy**: ETHOS.md documents that AI makes "completeness cheap" — the skills consistently choose to do the complete thing rather than a shortcut. This philosophical stance is baked into the preamble ("Boil the Lake").

6. **Host-agnostic distribution**: A single skill template generates valid, host-adapted prompts for Claude, Codex, Factory, OpenCode, and others — with per-host frontmatter transforms, path rewrites, and tool renames.

7. **Three-tier testing**: Free (validation, <2s), paid gate (LLM-judge, ~$4), paid periodic (full E2E, ~$3.85) — with diff-based selection so only affected skills are tested per PR.
