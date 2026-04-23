# _best-patterns-playbook.md

Best architectural patterns extracted from gstack, ready to apply in other AI agent systems.

---

## Reusable Architecture Ideas

### 1. Persistent Daemon for Expensive-to-Initialize Tools

**Pattern**: Wrap any tool with expensive initialization (Chromium, database, ML model) in a long-lived HTTP server. CLI connects per command.

**Implementation guide**:
- State file (`~/.myapp/server.json`) with pid, port, token
- Health check via HTTP GET /health (not process table)
- Lock file with `wx` flag for atomic startup serialization
- Binary version hash to detect stale server → auto-restart
- Idle timeout for auto-shutdown

**When to use**: Any AI agent tool where initialization time > 500ms and multi-step usage is common (databases, ML models, browser instances, language servers).

**When not to use**: Single-query use cases where statelessness is a feature (simpler, no daemon management).

---

### 2. Prompt Compilation Pipeline

**Pattern**: Generate LLM prompts from typed source code using a build step, rather than writing prompts inline or assembling at runtime.

**Implementation guide**:
- Source templates (`.tmpl`) with `{{PLACEHOLDER}}` markers
- Typed resolver functions (`Record<string, ResolverFn>`)
- Build step: template → resolve → write output
- CI freshness check: fail if generated file is out of date with template
- Token budget monitoring as a warning (not hard gate)
- Multi-target output: same template → multiple formats (different AI agents, different languages)

**When to use**: 5+ prompts that share sections; prompts that need host-specific variations; teams where prompt drift is a risk.

**When not to use**: Single-prompt applications; rapid prototyping phases; when build step overhead exceeds benefit.

---

### 3. Behavioral Tier System for Prompt Composition

**Pattern**: Define N tiers of behavioral contracts. Each skill declares its tier. Tier 1 = minimal; Tier N = fully instrumented. Tiers are additive (each inherits lower tiers).

**Implementation guide**:
```typescript
function generatePreamble(tier: 1 | 2 | 3 | 4, context: TemplateContext): string {
  const sections = [
    ...tier1Generators.map(g => g(context)),
    ...(tier >= 2 ? tier2Generators.map(g => g(context)) : []),
    ...(tier >= 3 ? tier3Generators.map(g => g(context)) : []),
    ...(tier >= 4 ? tier4Generators.map(g => g(context)) : []),
  ];
  return sections.filter(Boolean).join('\n\n');
}
```

**When to use**: Systems with many prompts that need different levels of self-regulation. Adding a new behavioral instruction to "all complex skills" should be a one-file change.

---

### 4. Physical Port Separation for Partial API Exposure

**Pattern**: Bind two separate TCP ports. Expose the restricted surface via external proxy. Never use header-based origin checks for security boundaries.

**Implementation guide**:
- Port A (local): full API surface, 127.0.0.1 only
- Port B (tunnel): restricted allowlist, 127.0.0.1 only, forwarded via reverse proxy
- Never check `x-forwarded-for` to gate features — it's forgeable
- Port is the boundary; only TCP connections on the right port get the right surface

**When to use**: Any situation where you want to expose a subset of a local API externally (remote collaboration, webhooks, shared tools).

---

### 5. Host Abstraction for Multi-Target Distribution

**Pattern**: Define a typed `HostConfig` interface. Each target AI agent platform is one config. The generation pipeline uses the config to transform prompt output.

**Config surface**:
```typescript
interface HostConfig {
  frontmatter: { mode: 'allowlist' | 'denylist'; keepFields?, stripFields?, extraFields? }
  pathRewrites: Array<{ from: string; to: string }>
  toolRewrites?: Record<string, string>  // "Bash" → "shell"
  suppressedResolvers?: string[]         // placeholders to skip for this host
}
```

**When to use**: AI agent platforms proliferating — when you need to distribute the same workflows to Claude, Codex, GPT, and custom agents.

---

## Reusable Workflow Ideas

### 6. Diff-Based Test Selection with Declared Dependencies

**Pattern**: Each test declares which source files it depends on (as glob patterns). CI runs only tests whose dependency set overlaps with the current diff.

**Implementation guide**:
```typescript
const TEST_DEPENDENCIES = {
  'ship-e2e': ['ship/**', 'scripts/resolvers/review.ts', 'test/helpers/session-runner.ts'],
  'review-e2e': ['review/**', 'scripts/resolvers/review.ts', 'scripts/resolvers/review-army.ts'],
};

function selectTests(changedFiles: string[]): string[] {
  return Object.entries(TEST_DEPENDENCIES)
    .filter(([, deps]) => deps.some(pattern => changedFiles.some(f => matchGlob(f, pattern))))
    .map(([name]) => name);
}
```

**When to use**: Any test suite where tests have significant per-run cost (API calls, E2E browser tests, slow integration tests).

---

### 7. Three-Tier Test Cost Architecture

**Pattern**: Classify tests into free (<2s), gate (~cost per run, required for merge), and periodic (weekly, quality benchmarks).

**Classification criteria**:
- Free: static analysis, schema validation, command registry checks
- Gate: safety guardrails, deterministic functional tests, LLM judge on changed skills
- Periodic: quality benchmarks, non-deterministic tests, external service tests

**When to use**: Any project with expensive automated testing. Prevents the all-or-nothing choice between "run everything and go broke" vs. "run nothing and miss regressions."

---

### 8. Fix-First Review Pattern

**Pattern**: Before surfacing questions to the user, auto-fix all mechanical, obviously correct improvements. Only ask about ambiguous decisions.

**Implementation**:
- Phase 1 (automatic): whitespace, imports, naming, obvious typos, standard security patterns
- Phase 2 (question gate): ambiguous architectural decisions, taste calls, irreversible changes
- Batch questions: never ask one at a time; compile all questions, ask once

**When to use**: AI review workflows where user fatigue from too many questions is a risk.

---

## Reusable Memory Patterns

### 9. File-Based Cross-Session Memory with Structured Log Format

**Pattern**: Persist agent learnings as JSONL files in `~/.myapp/learnings/`. Read on session start; append on session end.

**When vector DB is overkill**:
- Single user
- < 10K learning entries
- Retrieval is recency-based or keyword-searchable
- No need for semantic similarity

**When to upgrade to vector DB**:
- Multi-user (need user-scoped embeddings)
- > 100K entries (grep becomes slow)
- Semantic retrieval needed ("find learnings similar to this bug")

---

### 10. In-Context Scratchpad with File-Based Checkpointing

**Pattern**: Use a structured in-context scratchpad (like TodoWrite) as working memory. Periodically persist to disk for recovery.

**Implementation guide**:
- Every N turns: write current scratchpad to `~/.myapp/checkpoint.json`
- On session start: check for checkpoint file; offer to resume
- Context recovery instruction: if context is pruned, re-read checkpoint

**When to use**: Long-running agent workflows (>10 turns) where session restart is likely or desired.

---

## Reusable Safety Patterns

### 11. Ensemble ML + Deterministic Tripwire for Content Safety

**Pattern**: Use multiple ML classifiers for probabilistic detection. Use a deterministic mechanism (canary) as an override that catches what ML misses.

**Implementation**:
```
L1-L3: Rule-based filters (fast, cheap, catch obvious cases)
L4: ML classifier A (probabilistic, domain-specific)
L4b: ML classifier B (different model, cross-confirm)
L5: Canary (inject secret token; if it leaks, block deterministically)
L6: Ensemble vote (2-of-3 at threshold → block)
```

**Key insight**: Single-classifier defenses have FP/FN problems. Ensemble + deterministic tripwire eliminates most edge cases. The canary is your "always on" safety net.

---

### 12. Audit Log as Append-Only JSONL

**Pattern**: Security events go to `attempts.jsonl` — never a database, never a mutable log.

**Properties**:
- Append-only (append, never update)
- Structured (JSONL, not free text)
- Rate-limited writes (60/min cap)
- Rotation at size limit (10MB, 5 generations)
- Salted hash of sensitive data (not full URLs)
- Async writes (never block event loop)

**When to use**: Any system with security-relevant events (auth failures, injection attempts, rate limits).

---

## Reusable Developer Productivity Patterns

### 13. CHANGELOG as User-Facing Product Notes

**Pattern**: Lead every CHANGELOG entry with user impact (what they can now do), not implementation details. Internal changes go in a "For contributors" section at the bottom.

**Template**:
```markdown
## [X.Y.Z]

**Two-line headline that lands like a verdict.**

3-5 sentence lead paragraph. Specific, concrete.

### The N numbers that matter
| Metric | Before | After | Δ |
...

### What this means for [user type]
2-4 sentences connecting metrics to workflow change.

---

### Itemized changes
#### Added / Changed / Fixed / For contributors
```

---

### 14. Slop-Scan: Quality-Focused AI Code Review

**Pattern**: Run a code quality scanner that catches patterns where AI-generated code is genuinely worse (not just "looks AI-ish").

**What to catch**:
- Empty catches around recoverable operations (use typed exceptions or safe wrappers)
- Redundant `return await` without try/catch
- String-matching on error messages (brittle)
- Over-broad catch blocks on narrow operations

**What not to fix**:
- Best-effort cleanup paths (should swallow all errors)
- Extension catch-and-log (Chrome extensions crash on uncaught)
- Fire-and-forget operations (empty catch is correct if you truly don't care)

---

### 15. Live-Symlink Development Workflow

**Pattern**: For tools that install skills/plugins, symlink the development directory into the install location for instant feedback.

**Implementation**:
```bash
# Install dev version
ln -s /path/to/dev/repo ~/.myapp/skills/myplugin

# Now: edit source file → change is immediately live in all sessions
# Risk: half-written changes break all users of the plugin
# Mitigation: check for symlink before major refactors; rm and use stable install
```

**When to use**: Rapid iteration on prompt/skill content. Not for infrastructure changes.
