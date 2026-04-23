# 08-runbook-for-learning.md — gstack

## Prerequisites

```bash
# Required
bun --version       # Must be >= 1.0.0
git --version       # For diff-based test selection

# For E2E tests
echo $ANTHROPIC_API_KEY  # Must be set

# For Codex E2E
ls ~/.codex/        # Codex auth config
```

## 1. Explore the Directory Structure

```bash
cd /home/user/gstack

# See all skill directories
ls -d */ | head -50

# Count skill templates
find . -name "SKILL.md.tmpl" | wc -l

# See the preamble generators
ls scripts/resolvers/preamble/

# See all resolver files with sizes
ls -lh scripts/resolvers/*.ts | sort -k5 -rh

# Understand the host system
cat hosts/index.ts

# Check installed version
cat VERSION
```

**What to observe**: 38+ skill directories, ~42 .tmpl files, resolver file sizes (design.ts and review.ts are huge), 10 hosts, version 1.6.3.0.

## 2. Understand the Skill System

```bash
# Read a generated skill (what Claude actually sees)
cat ship/SKILL.md | head -150

# Compare with its template source
cat ship/SKILL.md.tmpl | head -100

# Find all {{PLACEHOLDER}} markers in the template
grep '{{' ship/SKILL.md.tmpl

# See what resolvers exist
cat scripts/resolvers/index.ts | head -80

# Check preamble tier declarations across skills
grep 'preamble-tier' */SKILL.md.tmpl 2>/dev/null | sort -t: -k3 -n
```

**What to observe**: SKILL.md is a compiled output. The preamble section (huge) was generated from resolvers. Placeholders like `{{PREAMBLE}}`, `{{BROWSE_SETUP}}`, `{{BASE_BRANCH_DETECT}}` map to specific resolver functions.

## 3. Rebuild a SKILL.md from Template

```bash
# Dry run — see what would be generated without writing
bun run gen:skill-docs -- --dry-run 2>&1 | head -50

# Generate for a specific skill
bun run gen:skill-docs -- --skill browse 2>&1

# Check if generated files are fresh vs templates
bun test test/gen-skill-docs.test.ts

# Check skill health dashboard
bun run skill:check
```

**What to observe**: The generator reads `.tmpl` files, expands placeholders, writes `.md` files. Watch for token warnings (>160KB). Skill check shows pass/fail for each skill.

## 4. Run the Free Tests (< 2s)

```bash
# Run all free tests
bun test

# Run just skill validation
bun test test/skill-validation.test.ts

# Run browse integration tests
bun test browse/test/
```

**What to observe**: Which tests pass/fail. Skill validation catches literal `{{PLACEHOLDER}}` strings in generated files, stale SKILL.md files, missing commands in the registry.

## 5. Start the Browse Daemon

```bash
# First, build the browse binary if not already built
bun run build   # or: cd browse && bun build --compile src/cli.ts --outfile dist/browse

# Set up the $B alias
alias $B="./browse/dist/browse"
# OR if installed globally
alias $B="browse"

# Start the daemon by running a command
$B goto https://example.com

# Observe the state file created
cat ~/.gstack/browse.json

# Check daemon health
$B status

# Take a snapshot
$B snapshot
$B snapshot -i  # interactive elements only

# Take a screenshot
$B screenshot --out /tmp/test-screenshot.png

# Stop the daemon
$B stop
```

**What to observe**: `browse.json` contains pid, port (random 10000-60000), token, startedAt. Snapshot output shows `@e1`, `@e2` refs. Screenshot creates PNG file.

## 6. Explore the Security Architecture

```bash
# Read the security layer
cat browse/src/security.ts | head -100

# Read content security (DOM stripping)
cat browse/src/content-security.ts | head -80

# Read tunnel denial log
cat browse/src/tunnel-denial-log.ts

# After running some commands, check if attack log exists
ls -la ~/.gstack/security/
cat ~/.gstack/security/session-state.json 2>/dev/null || echo "No session state yet"
```

**What to observe**: `THRESHOLDS` constants, `combineVerdict()` logic, the ARIA injection patterns list in content-security.ts.

## 7. Trace the Server Lifecycle

```bash
# Start daemon, observe log output
$B goto https://httpbin.org/get &
sleep 2
cat ~/.gstack/browse.json

# Check what port it's using
cat ~/.gstack/browse.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['port'])"

# Manually hit the health endpoint
PORT=$(cat ~/.gstack/browse.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['port'])")
curl -s http://127.0.0.1:$PORT/health

# Try a command via HTTP directly (educational only)
TOKEN=$(cat ~/.gstack/browse.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['token'])")
curl -s -X POST http://127.0.0.1:$PORT/command \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"cmd":"text","args":[]}'
```

**What to observe**: The HTTP API is simple JSON-over-HTTP. The token in `browse.json` is the Bearer auth. The port changes on each restart (random selection).

## 8. Understand Test Infrastructure

```bash
# See which E2E tests would run based on current diff
bun run eval:select

# List past eval runs
bun run eval:list

# See where evals are stored
ls ~/.gstack/projects/ 2>/dev/null || echo "No evals yet"

# Read the touchfiles to understand test selection
cat test/helpers/touchfiles.ts | grep -A5 '"ship"'   # ship test dependencies
cat test/helpers/touchfiles.ts | grep -A5 '"review"' # review test dependencies

# Understand E2E tier classification
grep -A2 'gate\|periodic' test/helpers/touchfiles.ts | head -40
```

**What to observe**: `E2E_TOUCHFILES` array maps test names to file globs. Gate tests run in CI; periodic tests run weekly. `eval:select` shows which tests would run for the current branch diff.

## 9. Explore the Host System

```bash
# Read all host configs
for h in hosts/*.ts; do
  echo "=== $h ==="; head -20 "$h"; echo
done

# Understand frontmatter transforms for Claude
cat hosts/claude.ts

# See how Codex differs
cat hosts/codex.ts

# Generate skills for a non-Claude host (dry run)
bun run gen:skill-docs -- --host codex --dry-run 2>&1 | head -30
```

**What to observe**: Each host has different `globalRoot`, `frontmatter.mode` (allowlist vs. denylist), `pathRewrites`, `toolRewrites`. Claude is `denylist` (keep most fields), others may be `allowlist` (explicitly keep only certain fields).

## 10. Read a Design Document

```bash
# List all design docs
ls docs/designs/

# Read the architecture rationale (short, high value)
cat ARCHITECTURE.md | head -120

# Read the browser daemon design
cat docs/designs/GSTACK_BROWSER_V0.md | head -80

# Read the prompt injection security design
cat docs/designs/ML_PROMPT_INJECTION_KILLER.md | head -80
```

**What to observe**: ARCHITECTURE.md explains the "why Bun" and "why persistent daemon" decisions concisely. Design docs show the thinking behind major features before they were built.

## 11. Trigger Important Workflows (Safe Experiments)

```bash
# Validate all skills without running LLM
bun test test/skill-validation.test.ts

# Check for slop in current diff
bun run slop:diff

# Run skill health dashboard
bun run skill:check

# Watch mode for skill development (auto-regen on change)
# (Run in separate terminal)
bun run dev:skill

# Preview eval selection for current branch
bun run eval:select

# List recent eval results
bun run eval:list
bun run eval:compare  # compare two most recent
```

## 12. Inspect Memory and State

```bash
# User config
cat ~/.gstack/config.yaml 2>/dev/null || echo "No config yet"

# Server state (if running)
cat ~/.gstack/browse.json 2>/dev/null || echo "Server not running"

# Security state
ls ~/.gstack/security/ 2>/dev/null || echo "No security state yet"

# Eval history
ls ~/.gstack/projects/ 2>/dev/null || echo "No eval history yet"

# Model cache (downloaded on first security classifier use)
ls ~/.gstack/models/ 2>/dev/null || echo "No model cache yet"

# Version tracking
cat ~/.gstack/.last-setup-version 2>/dev/null || echo "No version marker"
```

## 13. Observe Skill Behavior in Action

```bash
# Run a simple skill via claude -p (requires ANTHROPIC_API_KEY)
# This simulates what E2E tests do
echo "List the top 5 files by size in the current directory" | \
  claude -p --system "$(cat browse/SKILL.md)" \
  --output-format stream-json 2>&1 | head -50

# Or directly invoke a skill in Claude Code (interactive)
# claude
# /browse
# /review
```

## 14. Understand the Build Pipeline

```bash
# See what build does
cat package.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['scripts'].get('build',''))"

# Check binary sizes after build
ls -lh browse/dist/ 2>/dev/null || echo "Not built yet"

# Understand compilation
# browse/src/cli.ts → bun build --compile → browse/dist/browse (~58MB)
```

## Where Logs Appear

| Source | Log Location |
|--------|-------------|
| Browse daemon stdout | Terminal where `$B` was first run (or daemon stdout) |
| Security attacks | `~/.gstack/security/attempts.jsonl` |
| Eval results | `~/.gstack/projects/$SLUG/evals/` |
| GitHub Actions | `.github/workflows/` → GitHub Actions UI |
| Skill validation | `bun test` stdout |
| Build output | `bun run build` stdout |

## How to Safely Experiment Without Breaking the Repo

1. **Never `git add .` or `git add -A`** — binary files in `browse/dist/` and `design/dist/` will be staged accidentally
2. **Use `--dry-run` with gen-skill-docs** before writing generated files
3. **Don't edit SKILL.md directly** — edit SKILL.md.tmpl and regenerate
4. **Resolving SKILL.md merge conflicts**: always resolve .tmpl first, then regenerate
5. **Browse daemon experiments**: `$B stop` before experiments; always check `~/.gstack/browse.json` is cleaned up
6. **Don't run `bun run test:evals`** unless you have `ANTHROPIC_API_KEY` and are willing to spend ~$4
7. **Local skill symlink**: check `ls -la .claude/skills/gstack` — if it's a symlink, your changes go live immediately for all Claude Code sessions
