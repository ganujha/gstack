# 16-production-readiness-review.md — gstack

> Assessment assumes "production" means: (a) broad deployment across YC portfolio companies, with teams of 3-30 engineers per install, and (b) daily use of the full skill surface, including paid evals.
>
> For a single-developer install on one macOS laptop, gstack is already in high-confidence production. This review targets the "team + scale + fleet" case.

---

## Headline: Current Production Readiness

| Dimension | Single-user | Small team | YC-wide fleet |
|-----------|-------------|------------|---------------|
| Correctness | High | High | Medium |
| Reliability | High | Medium | Low-Medium |
| Security | High | High | Medium (no signed skills) |
| Observability | Low | Low | Low |
| Cost control | Medium | Low | Low |
| Multi-user sharing | N/A | Low | Low |
| Windows support | Medium | Medium | Medium |
| Operability | High | Medium | Low |
| Incident response | User-self-service | Limited | None documented |

Short version: gstack is a **robust single-user tool** and a **capable team tool with guardrails**. It is **not yet a fleet-ready product**. The gap is not in the architecture — it's in the controls, observability, and multi-tenant story.

---

## Prototype-grade vs Production-grade (what's which)

### Production-grade
- Browse daemon core (state atomicity, auth, idle shutdown, dual-listener, path+command allowlists).
- Skill template compile pipeline (typed resolvers, fail-fast, freshness CI).
- Diff-based test selection + worktree isolation.
- Security canary and combineVerdict ensemble.
- Atomic file writes for startup state.

### Production-capable but under-exercised
- ML security classifiers (documented but no red-team test suite).
- Multi-host skill distribution (tested on Claude; other hosts are "generated and hope").
- Eval store trend comparison (useful for single-user, no fleet rollup).
- pair-agent tunnel mode (works, but disconnect recovery is thin).

### Prototype-grade
- Observability (stdout logs + local JSONL + eval store; no trace correlation).
- Multi-workspace concurrency (one Chromium per user; tab contention on shared state).
- Telemetry (fire-and-forget, no retry, no central dashboard documented in repo).
- Windows support (a fallback exists; CI does not validate it).
- Long-running workflow recovery (no checkpoint on daemon crash).

### Experimental / aspirational
- GBrain integration (host config exists; runtime behavior not visible in repo).
- Conductor workspace concurrency claims ("10 workspaces, zero conflicts" — tested informally).
- DeBERTa ensemble (opt-in via env var; 721MB download; not in default flow).
- Community-pulse Supabase function (endpoint exists; consumption pattern unclear).

---

## Missing Controls

| Missing Control | Why It Matters | Quick Fix | Proper Fix |
|----------------|----------------|-----------|------------|
| Import boundary enforcement | `security-classifier.ts` into `server.ts` silently breaks compiled binary | Add `bun test` grep check | dependency-cruiser config |
| PR-wide eval budget cap | Single accidental "run all evals" costs $50-100 | Env var `GSTACK_MAX_PR_COST_USD` | Central budget daemon that aggregates across runs |
| Signed skill distribution | Supply-chain compromise of repo reaches all users | SHA-256 manifest of skill content | Signed releases + verification in `/gstack-upgrade` |
| Red-team test fixtures | No automated proof the ensemble blocks known attacks | Add 20 fixture payloads | Continuous red-team suite, quarterly updates |
| Pre-commit block on `dist/` | Binary-in-git rot persists | `pre-commit` hook | One-time `git rm --cached` + ignore |
| LLM judge determinism | Eval trends meaningless | `temperature: 0` | `temperature: 0` + structured output |
| Concurrent Claude session isolation | Tab + state contention | Per-session Playwright contexts | Session-scoped daemons |

---

## Missing Validation Paths

1. **No validation that `gstack-slug` returns the same slug across machines**. A team running evals on laptop and in CI could split history across two namespaces without noticing.
2. **No validation that a skill generated for Codex actually works on Codex**. The compile step produces valid-looking markdown; end-to-end verification is Claude-only.
3. **No validation of browser state cleanliness between skills**. Cookies, localStorage, IndexedDB, and tab state can leak across skill invocations.
4. **No validation of `--max-turns` being sufficient for ship-scale skills**. Tests use short fixtures; production skill runs may hit the turn cap.
5. **No validation of tunnel security under active attack**. The deterministic canary is tested implicitly; path+command allowlist is tested by unit, not by scripted attacker.
6. **No validation that the idle timer resets correctly in headed mode**. Documented as "headed mode never auto-dies"; not tested.
7. **No validation that atomic rename survives an ENOSPC disk-full mid-write**. Ends in a `.tmp` file and a missing real file.

---

## Missing Observability

### Low-effort additions
- **Request correlation IDs**: stamp every `/command` with a UUID, echo it in stdout log and in any outbound Supabase call.
- **Latency histograms in `/health`**: expose p50/p95/p99 of recent command durations.
- **Error rate counter**: rolling 60-second error count at `/health`.

### Medium-effort additions
- **Structured logger** (pino): replace scattered `console.log` with categorized events. Persist to a rolling file with rotation.
- **Eval summary upload**: opt-in nightly aggregation to Supabase for a cross-user "this skill broke for 11% of runs" view.
- **Bun binary version in state file**: already present. Surface it in telemetry for supply-chain auditability.

### High-effort additions
- **Distributed tracing across sidebar-agent + server**: trace IDs passed via session-state.json; spans persisted to a local SQLite for on-machine inspection.
- **Cost dashboard**: aggregate pricing.ts computations into a per-project, per-skill dashboard.

---

## Missing Security Hardening

1. **No signed releases**. `/gstack-upgrade` pulls from `main` and rebuilds. A compromised commit on main reaches every user.
2. **No secret scanning on skill templates**. A contributor who accidentally commits an API key in a template ships it.
3. **No rate limits on the `/command` HTTP surface**. Local only, so mostly fine, but a compromised shell on the host could spam the daemon.
4. **No Content-Security-Policy on the extension**. Standard extension hardening is elsewhere (manifest.json could enforce more).
5. **Prompt injection countermeasures don't cover the model's own output**. If the agent decides to write a malicious command to a file, L4/L4b don't gate write commands; they gate read commands.
6. **No auditable record of `GSTACK_SECURITY_OFF` usage**. Setting the env var disables defenses silently.
7. **`bin/gstack-*` shell utilities are varied in input validation quality**. They're called from skills; a skill that passes untrusted content (from a web page via $B) into a bin script is a shell-injection risk vector worth reviewing.

---

## Missing Operational Runbooks

The repo has `ARCHITECTURE.md` and `CLAUDE.md` but no explicit runbooks for:

- **Daemon won't start**. What are the common causes? lock file, port conflict, Playwright install issue.
- **ML classifier model download fails**. What does the user see? How do they retry?
- **Eval costs are unexpectedly high**. How to audit, how to cap next time.
- **Attack log is filling up**. How to rotate, how to disable, how to ship to a SIEM.
- **Windows-specific failure modes**. What differs from macOS/Linux.
- **ngrok disconnected mid-pair-agent**. Recovery path.
- **Multiple machines, same user, same repo**. Coordination story.

A one-page "What To Do When…" runbook per item would make support self-service.

---

## Missing Failure Handling

| Failure | Current behavior | Needed behavior |
|---------|------------------|-----------------|
| Browse daemon crash mid-skill | CLI gets ECONNREFUSED; one retry; then skill fails | Skill checkpoint + resume |
| Chromium OOM | Playwright throws; next command retries | Detect OOM explicitly; reduce concurrent pages |
| Supabase ingest 5xx | Silently drops event | Local buffer + retry with exponential backoff |
| 429 on Anthropic (judge / sidebar) | Single retry | Retry with backoff; surface to user after N failures |
| Disk full during state write | `.tmp` file stays; original intact | Explicit ENOSPC handling + user warning |
| Chromium subprocess orphaned | Idle-shutdown eventually reaps; possibly not | Parent-PID watchdog; cleanup on daemon start |
| Pre-commit hook failure during `/ship` | Commit blocked, branch dirty | Explicit "fix and retry" message with diagnostic |

---

## Missing Cost / Latency Guardrails

1. **No per-skill cost ceiling**. A runaway `/ship` with a 10,000-file diff could burn more than intended.
2. **No per-test timeout budget separate from `max-turns`**. Turns can be fast or slow; wall-time is what matters to CI.
3. **No concurrency cap on Agent-tool fanout**. `/review-army` could spawn N reviewers where N grows with skill complexity.
4. **No cost dashboard or weekly digest**. Users discover spend at invoice time.
5. **No cold-path elimination in production**. The first `$B` after boot pays the startup cost every session; no warm pool.
6. **No CDN-cached model download**. TestSavantAI + DeBERTa models are hundreds of MB and come from HuggingFace each install.

---

## Top 10 Upgrades for Robust Production Use

Ordered by (value × likelihood-of-hitting-the-problem) / (implementation cost):

1. **`temperature: 0` + structured output on LLM judge**. Cost: ~30 minutes. Value: eval trends become reliable.
2. **Per-PR eval budget kill switch**. Cost: ~2 hours. Value: eliminates runaway spend.
3. **Import-boundary tests for load-bearing invariants**. Cost: ~1 hour. Value: prevents silent binary breakage.
4. **Red-team fixture set for security ensemble**. Cost: ~1 day. Value: proves defenses work; regression coverage.
5. **Pre-commit block + one-time cleanup of binary-in-git debt**. Cost: ~30 minutes. Value: removes longstanding debt.
6. **Structured logger + rolling log file + `/health` latency histogram**. Cost: ~4 hours. Value: debuggable production issues.
7. **Per-Claude-session Playwright browser contexts**. Cost: ~1 day. Value: multi-session safety.
8. **Skill checkpoint/resume for `/ship`**. Cost: ~2 days. Value: long-running workflow survives interruptions.
9. **Signed skill release + verification in `/gstack-upgrade`**. Cost: ~3 days. Value: supply-chain defense.
10. **Multi-host runtime parity audit**. Cost: ~2 days. Value: either make Codex/Factory/etc. work, or stop generating skills for them. Reduces "works on Claude, broken silently elsewhere" surface.

Everything else — cost dashboards, distributed tracing, Windows CI, incident runbooks — is valuable but lower on the priority list than these ten.

---

## What Would Make gstack Safe To Deploy Fleet-Wide Tomorrow

Three changes unlock the YC-wide fleet case:

1. **Signed skills + verified upgrade path**. Without this, a supply-chain attack is plausible at scale.
2. **Per-team budget + telemetry dashboards**. Without this, a team's first "surprise bill" could sour adoption.
3. **Operational runbooks + first-line support doc set**. Without this, each team needs direct contact with gstack maintainers.

The rest (observability depth, multi-session isolation, Windows hardening) can come incrementally once the fleet case has a floor.
