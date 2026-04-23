# 13-runtime-behavior-deep-dive.md — gstack

> Scope: describe the real runtime lifecycle, not the inferred one. For each flow, identify where control branches, where state mutates, where retries happen, where errors propagate, and where a human can intervene. Runtime state (`~/.gstack/`) does not exist in this sandbox, so observations are code-traced rather than execution-confirmed. Where execution would change the interpretation, that is called out explicitly.

---

## Entry Path: How Control Enters the System

### Path A — User invokes a skill (`/ship`, `/review`, etc.)

```
1. Claude Code CLI is running
2. User types "/ship"
3. Claude Code locates ~/.claude/skills/gstack/ship/SKILL.md
4. Claude Code loads SKILL.md as system prompt for this turn
5. Claude begins executing the workflow steps in the prompt
6. Each Bash/Edit/Read/Agent tool call is invoked by the model
7. Any $B <cmd> invocation opens an HTTP connection to a locally running browse daemon (or starts one first)
```

Control enters entirely through Claude Code's built-in agent loop. **gstack provides no runtime itself; it provides inputs (prompts) and side-channel tools (browse daemon).**

### Path B — User runs `$B` directly

```
1. Shell expands $B → browse binary in PATH
2. browse cli.ts: parse argv, determine command
3. cli.ts: readState() → ~/.gstack/browse.json
4. If server is alive and healthy → POST /command with auth token
5. If not → ensureServer() → atomic lock → spawn detached Bun subprocess → wait for health → POST /command
```

### Path C — Test runner spawns `claude -p`

```
1. `bun test` discovers test files
2. E2E test constructs a prompt + calls session-runner.ts
3. session-runner spawns shell: cat promptFile | claude -p [flags]
4. claude binary runs with allowlisted tools, --output-format stream-json
5. NDJSON flows back via stdout → parseNDJSON → SkillTestResult
6. llm-judge.ts scores the result; eval-store.ts persists it
```

---

## Startup Lifecycle — Browse Daemon (authoritative trace)

Trace reconstructed from `browse/src/cli.ts:221-306` and `browse/src/server.ts:2712-2789`.

```
$B goto https://example.com
    │
    ▼
cli.ts: parse args
    │
    ▼
cli.ts: readState() → ~/.gstack/browse.json
    │
    ├── file missing ─────────────┐
    ▼                              ▼
cli.ts: isServerHealthy()      ensureServer()
  GET 127.0.0.1:PORT/health     (missing state case)
    │ 2s timeout
    ├── alive ────────► runCommand()  [POST /command]
    │
    └── dead ──────────► ensureServer()
                             │
                             ▼
                         acquireServerLock()
                           (atomic `fs.openSync(lockPath, 'wx')`)
                             │
                             ▼
                         startServer()
                           ├── macOS/Linux: Bun.spawn([bun, run, SERVER_SCRIPT]) + unref
                           └── Windows:     child_process.spawn(node, [server-node.mjs])
                             │
                             ▼
                         [subprocess begins]
                             │
                             ▼
                         server.ts: Bun.serve({port: random, hostname: 127.0.0.1})
                         server.ts: Playwright.launch() (Chromium subprocess)
                         server.ts: writeState() → atomic temp + rename
                             │
                             ▼
                         [parent polling loop]
                         while (elapsed < MAX_START_WAIT) {
                           readState()
                           isServerHealthy() → GET /health
                           Bun.sleep(100)
                         }
                         MAX_START_WAIT = Windows 15s | CI 30s | default 8s
                             │
                             ▼
                         releaseServerLock()
                             │
                             ▼
                         runCommand() → POST /command
```

### Where state mutates during startup

| State | Written by | Written how |
|-------|-----------|-------------|
| `~/.gstack/browse.json.lock` | cli.ts:288 `acquireServerLock` | Atomic `wx` open |
| `~/.gstack/browse.json` | server.ts:2728-2730 | `.tmp` write → `rename` |
| `Playwright browser user-data` | Chromium subprocess | Playwright-managed |
| `process.env.BROWSE_*` in subprocess | Bun.spawn env in cli.ts | In-process |

### Where branching happens

| Branch | Condition | File:line |
|--------|-----------|-----------|
| Server alive vs dead | health check | cli.ts:113-124 |
| macOS vs Windows | `IS_WINDOWS` constant | cli.ts:59-85, 221-232 |
| Version mismatch vs match | binaryVersion field compare | cli.ts (version gate) |
| Local vs tunnel surface routing | `surface` param in `makeFetchHandler` | server.ts:1564-1590 |

### Where retries happen

| Retry point | Policy |
|-------------|--------|
| Health check polling | 100ms interval, capped at MAX_START_WAIT |
| Single connection retry on ECONNREFUSED | cli.ts (one retry, then fail) |
| 429 in LLM judge | One retry with 1s delay |
| None | `claude -p` subprocess, ngrok tunnel disconnect, Supabase ingest |

### Where errors propagate

- **Startup failure** → CLI exits with descriptive message; lock file is released via a try/finally. Evidence: `cli.ts:283-306`.
- **Tool handler failure inside daemon** → returns JSON error body with `error: message`; daemon continues running.
- **Chromium crash** → Playwright throws; server attempts restart on next request (inferred from browser-manager.ts usage, not confirmed).
- **ML classifier failure** → `security-classifier.ts` likely degrades to WARN-only behavior (`GSTACK_SECURITY_OFF` equivalent) but this is Unverified.

### Where logging/telemetry occurs

- `console.log` / `console.warn` to subprocess stdout (captured by shell redirect only if the user sets up one; otherwise inherited by parent during startup, then lost after detach).
- Attack attempts → `~/.gstack/security/attempts.jsonl` (appendFile, no sync flush).
- Telemetry → `bin/gstack-telemetry-log` writes to a local batch; `gstack-telemetry-sync` pushes to Supabase edge function.
- Eval results → `~/.gstack/projects/$SLUG/evals/*.json`.
- **Observation**: no unified logging format, no rotation on most files except attack log, no trace IDs.

### Where humans can intervene

1. **Skill invocation**: Every skill is user-triggered.
2. **AskUserQuestion**: Inside a skill, the agent surfaces ambiguous decisions.
3. **Security kill switch**: `GSTACK_SECURITY_OFF=1` disables ML classifiers.
4. **Force-quit**: `kill $(cat ~/.gstack/browse.json | jq .pid)`.
5. **`./setup`**: Only runs when user invokes it; migrations in `gstack-upgrade/migrations/` run after.

---

## Request/Task Flow — `$B click @e3`

```
shell: $B click @e3
    │
    ▼
cli.ts: ensureServer() → POST /command
    │    body: { cmd: "click", args: ["@e3"], token: "..." }
    ▼
server.ts: fetch handler (surface='local')
    │
    ├── path !== /command → 404
    │
    ▼
server.ts: auth check (bearer token === AUTH_TOKEN)
    │ failure → 403
    ▼
server.ts: resetIdleTimer()
    │
    ▼
server.ts: handleCommand('click', ['@e3'])
    │
    ▼
write-commands.ts: click(ref='@e3')
    │
    ▼
browser-manager.ts: resolveLocatorFromRef('@e3') → Playwright Locator
    │
    ├── ref missing from map → throw "unknown ref"
    │
    ▼
page.locator.click({timeout: N})
    │
    ├── timeout / not-visible → Playwright throws
    │
    ▼
return JSON {ok: true, url: currentUrl}
    │
    ▼
HTTP 200 response → cli.ts → stdout → Claude's Bash output
```

### Invariants maintained by this path

- No CSS/XPath selector ever enters the agent's context. The ref-to-locator map is server-side.
- Every call resets the idle timer (server auto-shutdown defers by 30min on activity).
- Auth token is checked before any handler logic runs; no bypass for `/command`.

### What breaks this path

- **Stale ref**: Agent uses `@e3` after navigation. The locator map still exists but the element does not. Playwright throws. This surfaces as an error string to the agent, which may or may not re-snapshot. There is **no automatic snapshot-before-write** enforcement.
- **Concurrent CLI**: Two shell invocations hitting the same server at the same time race through `handleCommand`. Playwright serializes page ops internally, but command ordering is not deterministic.
- **Server dies mid-request**: Client sees `ECONNREFUSED` → one retry. If retry also fails, the skill workflow fails.

---

## Request Flow — Sidebar Chat (headed mode)

```
Chrome extension sidepanel → POST /sidebar-chat
    │
    ▼
server.ts: auth via gstack_sse cookie (30-min HttpOnly, set by POST /sse-session)
    │
    ▼
server.ts: pickSidebarModel(userMessage)
    │ regex routing: Opus for analysis, Sonnet for actions
    ▼
[content goes through L1-L3 transforms on server-side]
    │
    ▼
server.ts: dispatch to sidebar-agent.ts subprocess via IPC/state file
    │
    ▼
sidebar-agent.ts:
    ├── L4:  classifyContent(html, via ONNX)    [~10-50ms]
    ├── L4b: checkTranscript(conversation, via Haiku)  [~200ms + API cost]
    ├── L5:  canary check on page content (deterministic)
    └── L6:  combineVerdict(verdict1, verdict2, canary) → SAFE | WARN | BLOCK
    │
    ├── BLOCK → append to attempts.jsonl → return blocked message
    │
    ▼
sidebar-agent.ts: invoke Anthropic SDK with model from router
    │
    ▼
response → SSE back to sidepanel
```

**Observation**: The sidebar path is where the full 6-signal stack actually executes. The plain `$B <cmd>` path runs only L1-L3 (content transforms) on response content; the heavy ML layers are sidebar-specific.

---

## Skill Execution Flow (from Claude's POV)

For a skill like `/ship`:

```
Turn 1: Load SKILL.md (preamble + skill body), TODOS.md, CHANGELOG.md, VERSION
Turn 2: Read git status + recent commits
Turn 3: Run `bun test` via Bash (blocks until exit)
Turn 4: Decide next step from SKILL.md's 19 steps
...
Turn N: Ask user (AskUserQuestion) if version bump type is ambiguous
...
Turn N+k: Write CHANGELOG.md, VERSION, commit, push
Turn N+k+1: Spawn canary subagent via Agent tool (if skill calls for it)
```

### What Claude does NOT know

- That the browse daemon has a 30-minute idle timeout (Claude does not hold the daemon open).
- That concurrent skill sessions can race on shared state files.
- That `claude -p` subprocesses running tests are using up budget.

### Recovery behavior

- **`generate-context-recovery.ts`**: prompts Claude to re-read TodoWrite if context was compressed.
- **`generate-continuous-checkpoint.ts`**: prompts Claude to write progress markers to disk periodically.
- **`generate-confusion-protocol.ts`**: tells Claude to stop and AskUserQuestion rather than hallucinate progress.

None of these are code-enforced. They are **prompt-level instructions**, which means Claude can ignore them under context pressure.

---

## Control Plane vs Data Plane

**Control plane** (decides what happens): Claude + skill templates. Everything important is a prompt.

**Data plane** (executes and records): the browse daemon, file system, Supabase telemetry. These have small amounts of imperative code that never makes policy decisions — they only enforce transport-layer and auth-layer invariants.

This split is unusual. Most agent systems have a Python/TS orchestrator that makes decisions and calls LLMs as subroutines; here, the LLM is the orchestrator and the code is its I/O library. A second-pass reader should internalize this framing: **policy = prompts, mechanism = code**. The resolver library is the compile step that turns policy into prompts.

---

## Determinism Inventory

| Operation | Deterministic? | Why |
|-----------|---------------|-----|
| State file atomic write | Yes | `.tmp + rename` |
| Startup lock acquisition | Yes | `wx` flag |
| Auth token comparison | Yes | String equality |
| Path allowlist (tunnel) | Yes | Set membership |
| Command allowlist (tunnel) | Yes | Set membership |
| Idle-shutdown decision | Yes | timestamp comparison |
| Canary injection/detection | Yes | Random-string + exact substring match |
| combineVerdict thresholds | Yes | Score comparisons (but input scores are stochastic) |
| `pickSidebarModel` | Yes | Regex match |
| Placeholder expansion | Yes | Single-pass regex + fail-fast |
| Diff-based test selection | Yes | Glob match on `git diff --name-only` |
| Touchfile-to-test mapping | Yes | Static record lookup |
| LLM judge scores | **No** | Default temperature; regex JSON extraction |
| ML classifier verdicts | **No** | Model inference |
| Claude's choice of next step in a skill | **No** | Model inference |
| Agent-tool subagent fanout | **No** | Each subagent makes its own decisions |

**Implication**: Every score that is meant to be "compared across runs" needs to be interpreted as a distribution, not a point estimate. Eval trends are signals, not facts.

---

## What Was Verified By Reading Actual Code This Pass

| Subsystem | File:line | Claim it confirmed |
|-----------|-----------|---------------------|
| Dual-listener | server.ts:2712, 1907, 2785 | Two `Bun.serve` calls, separate ports |
| Idle shutdown | server.ts:63, 899-909 | 30-min default, 60s check cadence, headed/tunnel exempt |
| Atomic state write | server.ts:2728-2730 | `.tmp + rename`, 0o600 mode |
| Startup lock | cli.ts:283-306 | `fs.openSync(..., 'wx')` |
| Windows fallback | cli.ts:59-85, build-node-server.sh | `server-node.mjs` artifact |
| Sidebar model routing | server.ts:190-211 | 3 regexes, default Opus |
| Preamble tiers | preamble.ts:71-103 | Additive via spread |
| Placeholder expansion | gen-skill-docs.ts:430-448 | Single-pass, fail-fast |
| Resolver registry | resolvers/index.ts:26 | Static `Record<string, ResolverFn>` |
| Token ceiling | gen-skill-docs.ts:544-547 | Warning only |
| Touchfile selection | touchfiles.ts:34-510 | Global touchfiles trigger all |
| Session runner | session-runner.ts:119-366 | `claude -p` via shell, NDJSON parse |
| LLM judge | llm-judge.ts:40-192 | `claude-sonnet-4-6`, regex extraction, no temperature |
| Eval store | eval-store.ts:23-318 | project-scoped dir + slug fallback |
| Pricing | pricing.ts:20-61 | dynamic per-run computation |
| Worktree dedup | worktree.ts:75-230 | SHA256 hash index |
| Supabase telemetry | telemetry-ingest/index.ts:47-55 | anon key + RLS |

---

## What Remains Unverified

| Item | Why unverified | What would verify it |
|------|---------------|----------------------|
| `~/.gstack/browse.json` contents at runtime | Sandbox has no state dir | Run `$B goto https://example.com` on a machine with gstack installed |
| Idle shutdown triggers cleanly | Requires real daemon | Start daemon, wait 30 min, check for process |
| Version mismatch restart path | Requires rebuilt binary | Install new version over running daemon |
| ML classifier cold start | Requires empty model cache | First-run with `rm -rf ~/.gstack/models/` |
| Chromium crash recovery | Requires crash injection | `kill -9` the Chromium subprocess mid-request |
| Concurrent CLI race | Requires two shells | Run `$B <cmd>` twice in parallel |
| Eval trend comparison | Requires history | Run evals twice, invoke `bun run eval:compare` |

### Blockers preventing execution in this sandbox

1. Claude Code binary not available (cannot run `/ship`).
2. No `~/.gstack/` state directory; running `$B` would trigger first-run setup.
3. Playwright Chromium not installed (would need `bunx playwright install`).
4. `ANTHROPIC_API_KEY` not set in sandbox.
5. No GitHub remote configured for the branch.
