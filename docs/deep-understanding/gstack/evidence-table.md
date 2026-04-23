# evidence-table.md — gstack Repository

| Claim | Evidence File | Confidence | Verification Step |
|-------|--------------|------------|-------------------|
| Runtime is Bun >= 1.0.0 | `package.json` → `"engines": {"bun": ">=1.0.0"}` | HIGH | `bun --version` |
| Browse daemon is a persistent HTTP server | `browse/src/server.ts` → `Bun.serve()` | HIGH | Run `$B goto url`, check `~/.gstack/browse.json` pid |
| Daemon uses dual-listener topology (local + tunnel) | `ARCHITECTURE.md` §dual-listener, `browse/src/server.ts` | HIGH | Run `browse/src/server.ts` and inspect bound ports |
| Server state stored in `~/.gstack/browse.json` | `browse/src/cli.ts` `readState()` | HIGH | `cat ~/.gstack/browse.json` after start |
| Skills are generated from `.tmpl` templates | `scripts/gen-skill-docs.ts`, any `*/SKILL.md.tmpl` | HIGH | `bun run gen:skill-docs --dry-run` |
| 10 AI agent hosts supported | `hosts/index.ts` exports array | HIGH | Read `hosts/index.ts` |
| Preamble has 4 tiers | `scripts/resolvers/preamble.ts`, SKILL.md.tmpl frontmatter `preamble-tier:` | HIGH | Read preamble.ts tier switch |
| Security uses ML classifiers (TestSavantAI ONNX) | `browse/src/security.ts`, CLAUDE.md §sidebar security stack | HIGH | Check `~/.gstack/models/testsavant-small/` path |
| Eval results persist to `~/.gstack/projects/$SLUG/evals/` | `test/helpers/eval-store.ts` | HIGH | Run `bun run eval:list` |
| Diff-based test selection via touchfiles | `test/helpers/touchfiles.ts` `E2E_TOUCHFILES` array | HIGH | Run `bun run eval:select` |
| `session-runner.ts` spawns `claude -p` subprocess | `test/helpers/session-runner.ts` | HIGH | Read session-runner.ts spawn call |
| Snapshot uses Playwright ARIA tree | `browse/src/snapshot.ts` → `page.locator().ariaSnapshot()` | HIGH | `$B snapshot` and inspect output format |
| Canary injection is L5 deterministic check | `browse/src/security.ts` §THRESHOLDS, CLAUDE.md L5 | HIGH | Read security.ts combineVerdict |
| CHANGELOG.md is 321K | File size check by explore agent | HIGH | `wc -c CHANGELOG.md` |
| Version is 1.6.3.0 | `package.json` version field, `VERSION` file | HIGH | `cat VERSION` |
| Browse commands: 13 read, 20 write, 17 meta | `browse/src/commands.ts` READ_COMMANDS, WRITE_COMMANDS, META_COMMANDS | HIGH | Read commands.ts |
| 9 snapshot flags | `browse/src/snapshot.ts` SNAPSHOT_FLAGS array | HIGH | `$B snapshot --help` |
| macOS ARM64 requires codesign after build | `setup` script lines (codesign step) | HIGH | Inspect setup script codesign block |
| Windows uses Node.js fallback for server | `browse/src/cli.ts` Windows path (`server-node.mjs`) | HIGH | Read cli.ts Windows branch |
| 38 skill directories exist | Directory listing | HIGH | `ls -d */ | wc -l` at gstack root |
| `review.ts` resolver is 54K | File size from explore agent | HIGH | `wc -c scripts/resolvers/review.ts` |
| `design.ts` resolver is 58K | File size from explore agent | HIGH | `wc -c scripts/resolvers/design.ts` |
| Token ceiling warning at 160KB (~40K tokens) | `CLAUDE.md` §token ceiling, `gen-skill-docs.ts` | HIGH | Read gen-skill-docs.ts warning logic |
| Security attack log at `~/.gstack/security/attempts.jsonl` | `browse/src/tunnel-denial-log.ts` | HIGH | Read tunnel-denial-log.ts appendFile path |
| Rate limit: 60 tunnel denial writes/min | `browse/src/tunnel-denial-log.ts` sliding window | HIGH | Read rate limit constant in file |
| `lib/worktree.ts` manages git worktrees | `lib/worktree.ts` WorktreeManager class | HIGH | Read worktree.ts |
| LLM judge uses claude-sonnet-4-6 | `test/helpers/llm-judge.ts` model param | HIGH | Read llm-judge.ts model name |
| Judge scores: clarity, completeness, actionability (1-5) | `test/helpers/llm-judge.ts` JudgeScore interface | HIGH | Read JudgeScore type def |
| Playwright version: 1.58.2 | `package.json` dependencies | HIGH | `bun pm ls | grep playwright` |
| Anthropic SDK version: 0.78.0 | `package.json` dependencies | HIGH | `bun pm ls | grep anthropic` |
| 22 preamble generators in preamble/ subdirectory | `scripts/resolvers/preamble/` directory listing | HIGH | `ls scripts/resolvers/preamble/ | wc -l` |
| Extension is Chrome MV3 | `extension/manifest.json` (inferred from service worker) | MEDIUM | `cat extension/manifest.json` |
| sidebar-agent.ts exists for extension chat | CLAUDE.md §sidebar architecture, `browse/src/server.ts` mentions | MEDIUM | `find . -name "sidebar-agent.ts"` |
| `gstack-slug` derives project slug for eval namespacing | `test/helpers/eval-store.ts` references gstack-slug | MEDIUM | Run `bin/gstack-slug` in a project |
| Supabase powers analytics/telemetry backend | `supabase/functions/`, `bin/gstack-telemetry-sync` co-location | MEDIUM | Read supabase/functions/ contents |
| `gbrain` is an external agent runtime | `hosts/gbrain.ts`, `scripts/resolvers/gbrain.ts` | MEDIUM | Read gbrain.ts host config fields |
| `conductor.json` is for Conductor AI product | `conductor.json` at root, `gstack-slug` utility | LOW | Read conductor.json + search Conductor docs |
| Learnings persist cross-project for reuse | CLAUDE.md §learnings, `scripts/resolvers/learnings.ts` | MEDIUM | Read learnings.ts resolver |
| Device salt is 0600 permissions | CLAUDE.md §env knobs "device-salt" | MEDIUM | `ls -la ~/.gstack/security/device-salt` |
| Browse binary is ~58MB | CLAUDE.md "~58MB each" | MEDIUM | `ls -lh browse/dist/browse` |
| Codex auth from `~/.codex/auth.json` | `codex/SKILL.md.tmpl` auth probe section | HIGH | Read codex skill auth probe block |
| `pair-agent` generates instruction block for remote agents | `browse/src/cli.ts` pair-agent command | HIGH | Run `$B pair-agent` |
| Security kill switch via GSTACK_SECURITY_OFF=1 | CLAUDE.md §env knobs | HIGH | Set env var, watch classifier skip |
| Model cache rotates at 10MB (5 generations) | CLAUDE.md §attack log description | MEDIUM | Check `~/.gstack/security/` directory |
| Skill health check dashboard via `bun run skill:check` | `scripts/skill-check.ts`, package.json script | HIGH | Run `bun run skill:check` |
| Ship skill has 19 steps | `ship/SKILL.md.tmpl` section headers | HIGH | Read ship SKILL.md.tmpl |
| Auto-plan sequential: CEO → Design → Eng → DX | `autoplan/SKILL.md.tmpl` | HIGH | Read autoplan template |
| Office-hours is YC diagnostic (anti-sycophancy rules) | `office-hours/SKILL.md.tmpl` | HIGH | Read office-hours template |
| Investigate has 3-strike rule before escalation | `investigate/SKILL.md.tmpl` | HIGH | Read investigate template |
| Canary alerts: CRITICAL/HIGH/MEDIUM/LOW hierarchy | `canary/SKILL.md.tmpl` | HIGH | Read canary template |
| `slop-scan.config.json` only ignores vendor/ | File content `{"ignores": ["**/vendor/**"]}` | HIGH | `cat slop-scan.config.json` |
