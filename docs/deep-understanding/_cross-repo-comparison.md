# _cross-repo-comparison.md

> Note: This learning package covers a single repository (gstack). This document provides a comparative analysis of gstack's architecture against common patterns in the AI agent systems landscape, plus internal architectural comparisons across gstack's subsystems.

## Internal Subsystem Comparison (gstack)

### Agent Architecture

| Subsystem | Agent Pattern | Autonomy Level | Human Control |
|-----------|--------------|----------------|---------------|
| `/ship` skill | Single coordinator, optional canary subagent | High (19 steps auto) | VERSION bump, failing tests |
| `/autoplan` skill | Sequential multi-agent (CEO→Design→Eng→DX) | Medium (each phase gates) | Taste and User Challenge decisions |
| `/review` skill | Coordinator + parallel specialist fanout | Medium | fix-first, then ask |
| `/investigate` skill | Single agent with freeze boundary | Low (3-strike rule) | Every hypothesis before fix |
| Browse daemon | Tool server (not an agent) | N/A | Every command is human-or-agent-triggered |

**Pattern insight**: gstack's skills deliberately modulate autonomy. High-confidence, mechanical operations (git commit, CHANGELOG format) are auto-decided. Architectural decisions always surface to the human. This is a principled model: determinism for well-defined operations, escalation for judgment calls.

### Memory Design

| Layer | Approach | Scope | Durability |
|-------|----------|-------|-----------|
| In-context (TodoWrite) | Prompt-guided scratchpad | Session | Ephemeral |
| File system (TODOS.md, etc.) | Direct file writes | Project | Permanent |
| Learnings store | File-based log | Cross-project | Permanent |
| Eval store | Structured JSON | Per-project | Permanent |
| Security state | JSON (cross-process) | Session | Short-lived |

**Contrast with typical RAG systems**: No vector database, no semantic search over memory. gstack uses filesystem + grep for memory retrieval. This is a deliberate simplicity choice: easier to debug, no infrastructure dependency, appropriate for single-user tool.

### Tool Design

| Tool Surface | Abstraction Level | Tokens Used | Error Surface |
|--------------|------------------|-------------|---------------|
| $B @ref-based commands | High (refs hide selectors) | Low | Low (no CSS/XPath errors) |
| $B text/snapshot commands | Medium (returns page text) | Medium | Low |
| Bash (git, file ops) | Raw (shell) | Variable | High (any shell error) |
| Agent tool (subagent spawn) | High (prompt-only) | High (full context) | Medium |

**Key insight**: The @ref abstraction is gstack's biggest UX innovation for browser automation. By hiding CSS selectors behind short numeric refs, the agent's tool calls are shorter, more readable, and less error-prone.

---

## gstack vs. Common AI Agent Patterns

### vs. LangChain-style Tool Calling

| Dimension | gstack | LangChain |
|-----------|--------|-----------|
| Tool definition | HTTP command to persistent daemon | Python class with `run()` method |
| Tool invocation | Bash: `$B click @e3` | `tool.run(action)` in agent loop |
| Agent loop | External (Claude Code) | Framework-managed |
| Memory | Files + JSONL | Vector store (optional) |
| Multi-agent | Prompt-instructed (Agent tool) | Explicit graph (LangGraph) |

**gstack's advantage**: No framework lock-in. Skills work with any Claude Code session, with no Python imports. Tools are HTTP calls, not Python objects.

### vs. AutoGPT/BabyAGI Style

| Dimension | gstack | AutoGPT |
|-----------|--------|---------|
| Task planning | In-skill prompt specification | LLM-generated task list |
| Memory | Files + structured JSON | Vector embeddings |
| Agent loop | Claude Code (external) | Framework loop |
| Reliability | High (deterministic steps specified) | Lower (emergent planning) |
| Human control | Many structured pause points | Minimal |

**gstack's advantage**: Deterministic step specification beats emergent planning for well-understood workflows. You don't want `/ship` to invent its own shipping steps.

### vs. OpenAI Assistants / GPTs

| Dimension | gstack | OpenAI Assistants |
|-----------|--------|------------------|
| Skill format | Markdown file in directory | OpenAI platform configuration |
| Browser tool | Custom persistent daemon | Web browsing (opaque) |
| Security | 6-layer injection defense | Platform-managed (opaque) |
| Cost | Pay per eval run | Pay per API call |
| Portability | Multi-host (10 agents) | OpenAI-only |

**gstack's advantage**: Open, inspectable, portable. You own the skill prompts. You own the browser. You own the security model.

---

## Best Interview Stories Each Topic Enables

| Topic | Best Story |
|-------|-----------|
| **Persistent daemon** | "We cut per-command latency from 2-3s to ~100ms by keeping Chromium warm — 20x improvement for multi-step workflows" |
| **Prompt compilation** | "We built a TypeScript-to-markdown compiler for AI prompts — type safety at build, 10-host output from one template" |
| **Security ensemble** | "Stack Overflow false positives taught us: single ML layer isn't enough. 2-of-3 agreement + canary = robust injection defense" |
| **Diff-based test selection** | "We reduced eval costs by 80% by declaring per-test file dependencies and skipping tests when nothing they depend on changed" |
| **@ref abstraction** | "Hiding CSS selectors behind session-scoped refs reduced agent token use and eliminated an entire class of selector-syntax errors" |
| **Preamble tiers** | "40 skills share behavioral contracts via tier inheritance — adding confusion protocol to all Tier-2 skills required one file change" |
| **Boil the Lake** | "AI makes completeness cheap — our principle is to always do the full thing, not shortcuts. The /ship skill has 19 steps for a reason" |

---

## Production Readiness Comparison (gstack Subsystems)

| Subsystem | Readiness | Key Gap |
|-----------|-----------|---------|
| Browse daemon | High | Windows CI coverage, binary-in-git debt |
| Skill system | High | Non-Claude host test coverage |
| Security layer | High | No automated red-team test suite |
| Test infrastructure | High | Cost caps per test not enforced |
| State management | Medium | No atomic writes for browse.json, no file locking for CHANGELOG/TODOS |
| Multi-user support | Low | Single `~/.gstack/` per machine |
| Windows support | Medium | Node.js fallback exists, CI doesn't validate |
