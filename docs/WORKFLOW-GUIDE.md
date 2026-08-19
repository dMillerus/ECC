# ECC Workflow Guide

What you can actually run in this repo, and why you'd reach for it. Ordered from
"first hour" to "rarely, but you'll be glad it exists."

Two audiences share this repo and they run different things:

- **Users** installing ECC into a harness (Claude Code, Cursor, Codex, …) — sections 1–8.
- **Contributors** working on the repo itself — sections 9–11.

Commands are written to be run from a clone at the repo root unless stated otherwise.

---

## 1. Check prerequisites

```bash
claude --version   # need v2.1.0 or later
node --version     # need v18 or later
```

Run this before anything else. ECC's hook loading depends on Claude Code v2.1+ behavior,
and every script in `scripts/` is plain CommonJS Node.

---

## 2. Install ECC — pick exactly one path

The single most common broken setup is stacking the plugin install and the manual
installer. Pick one lane and stay in it.

### Path A — Claude Code plugin (recommended)

```
/plugin marketplace add https://github.com/affaan-m/ECC
/plugin install ecc@ecc
```

Run these inside a Claude Code session. This installs agents, skills, commands, and hooks.
Afterwards, commands are namespaced: `/ecc:plan "Add user authentication"`.

**Rules are not distributed by plugins** — copy only the rule packs you want:

```bash
git clone https://github.com/affaan-m/ECC.git
cd ECC
mkdir -p ~/.claude/rules/ecc
cp -R rules/common ~/.claude/rules/ecc/
cp -R rules/typescript ~/.claude/rules/ecc/   # add only the stacks you use
```

Copy the whole language directory, not the files inside it — relative references break otherwise.

### Path B — manual installer (finer control, no plugin)

```bash
git clone https://github.com/affaan-m/ECC.git
cd ECC
./install.sh --profile minimal --target claude
```

```powershell
.\install.ps1 --profile minimal --target claude
```

`install.sh` auto-runs `npm install` on first use if `node_modules/` is missing, then delegates
to `scripts/install-apply.js`. Profiles, smallest first: `minimal`, `core`, `security`, `developer`,
`research`, `full`.

- `minimal` deliberately excludes the hook runtime — use it if global hooks feel too invasive.
- `full` installs all 23 modules. Do not run it after a plugin install.

Other targets: `claude-project`, `cursor`, `antigravity`, `codex`, `gemini`, `opencode`,
`codebuddy`, `joycode`, `qwen`, `zed`.

### Preview before you commit to it

```bash
node scripts/install-apply.js --profile minimal --target claude --dry-run
```

Prints the resolved plan and operation count without copying a byte. Worth running every time.

### Add hooks later, or install only hooks

```bash
./install.sh --target claude --modules hooks-runtime
```

Writes resolved hooks to `~/.claude/hooks/hooks.json` and leaves `~/.claude/settings.json` alone.
Do **not** hand-copy the repo's `hooks/hooks.json` into `settings.json` — on a plugin install that
double-loads and executes hooks twice.

---

## 3. Figure out what to install

```bash
node scripts/consult.js "security reviews" --target claude
```

Natural-language advisor. Returns matching components, the profile they belong to, and a
ready-to-paste install and preview command for each. The fastest way in when you don't know
whether you want a skill, an agent, or a whole capability pack.

Browse the catalog directly:

```bash
node scripts/catalog.js profiles
node scripts/catalog.js components --family language
node scripts/catalog.js show framework:nextjs
```

Resolve a plan without installing:

```bash
node scripts/install-plan.js --profile core --target cursor
node scripts/install-plan.js --list-components --family capability --json
```

> The plan output carries its own caveat: target filtering reflects "scaffold-level adapter
> planning," not a byte-for-byte mirror of the legacy `install.sh` copy paths. Use
> `install-apply.js --dry-run` when you need the exact file operations.

Install one narrow thing instead of a profile:

```bash
node scripts/install-apply.js --profile minimal --target claude --with capability:security
node scripts/install-apply.js --target claude --skills continuous-learning-v2
```

---

## 4. Verify, repair, or back out an install

```bash
node scripts/ecc.js list-installed
node scripts/ecc.js doctor
node scripts/ecc.js repair --dry-run
node scripts/ecc.js repair
```

Reach for this when ECC looks duplicated, half-present, or after your Claude config was reset —
before reinstalling anything. `doctor` reports drift against the recorded install-state;
`repair` restores it.

Uninstall:

```bash
node scripts/uninstall.js --dry-run
node scripts/uninstall.js
```

Only removes files recorded in ECC's install-state. If you stacked install methods, unwind in
this order: remove the plugin → run uninstall → delete rule folders you copied by hand →
reinstall once, one path.

Update an existing install in place:

```bash
node scripts/ecc.js auto-update --dry-run
node scripts/ecc.js auto-update
```

Pulls the latest repo changes and reinstalls the managed targets using the original install request.

---

## 5. Daily use inside a session

These are slash commands, typed in your harness — not shell commands. Bare form after a manual
install (`/plan`), namespaced after a plugin install (`/ecc:plan`).

```
/plan "Add user authentication"     # plan first, waits for your confirm before touching code
/code-review                        # quality + security review of changed files
/build-fix                          # detect build system, route to the right build-resolver agent
/quality-gate                       # check against project standards
/test-coverage                      # coverage report + gap analysis
```

Language-specific variants follow a consistent `<lang>-<verb>` shape and exist for Go, Rust,
Kotlin, C++, Python, React, and Flutter — e.g. `/go-test`, `/rust-review`, `/cpp-build`,
`/react-test`, `/python-review`, `/flutter-build`.

Discovery, when you don't remember what exists:

```
/ecc-guide                # conversational map of the live repo surface
/ecc-guide find: <query>
/plugin list ecc@ecc      # plugin installs only
```

`/ecc-guide` reads the current checkout rather than hardcoded counts — prefer it over
`COMMANDS-QUICK-REF.md`, which is out of date (see section 14).

---

## 6. Session persistence and learning

```
/save-session       # write current session state to ~/.claude/session-data/
/resume-session     # reload the most recent saved session
/sessions           # browse, search, alias session history
/checkpoint         # mark a checkpoint mid-session
/context-budget     # find token overhead   (retired shim — see section 14)
```

Reach for `/save-session` before you close a long working session, `/resume-session` the next morning.

Pattern extraction — run at the end of a session that taught you something:

```
/learn-eval         # extract patterns + self-evaluate before saving
/instinct-status    # show learned instincts with confidence scores
/evolve             # cluster related instincts into skill structures
/instinct-export    # share your instincts
/instinct-import <file>
/skill-create       # analyze local git history -> generate a reusable skill
```

Inspect skill health from the shell:

```bash
node scripts/skills-health.js --dashboard
node scripts/skills-health.js --json
```

Backs the `/skill-health` command. Works from a clone; see the flag in section 14 about npm installs.

---

## 7. Security scanning

```bash
npx ecc-agentshield scan
npx ecc-agentshield scan --fix
npx ecc-agentshield scan --opus --stream
```

Scans `CLAUDE.md`, `settings.json`, MCP configs, hooks, agent definitions, and skills for secrets,
broad permissions, hook injection, and risky MCP servers. Exit code 2 on critical findings, so it
drops into CI as a gate. `--fix` applies only changes marked safe and auto-fixable.
`/security-scan` runs the same engine from inside a session.

AgentShield is a **separate npm package** (`ecc-agentshield`), not part of this install — `npx`
fetches it on demand.

Supply-chain IOC scan of your own dependency and AI-tool persistence surfaces:

```bash
node scripts/ecc.js security-ioc-scan --home
```

---

## 8. Configuration knobs

Package manager (npm / pnpm / yarn / bun):

```bash
node scripts/setup-package-manager.js --detect
node scripts/setup-package-manager.js --project bun
```

Detection order: `CLAUDE_PACKAGE_MANAGER` env var → `.claude/package-manager.json` in the project →
`packageManager` field in `package.json` → lockfile → `~/.claude/package-manager.json` → npm.
`/setup-pm` does the same thing from a session.

Hook runtime controls — tune strictness without editing hook files:

```bash
export ECC_HOOK_PROFILE=standard                 # minimal | standard | strict
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
export ECC_SESSION_START_MAX_CHARS=4000          # cap SessionStart context (default 8000)
export ECC_SESSION_START_CONTEXT=off             # disable SessionStart context entirely
export ECC_CONTEXT_MONITOR_COST_WARNINGS=off     # keep warnings, drop cost estimates
```

Skip bundled MCP servers you already run yourself:

```bash
export ECC_DISABLED_MCPS="github,context7,exa,playwright,sequential-thinking,memory"
```

This is an ECC install/sync filter, not a live Claude Code toggle — use `/mcp` for runtime changes.

API keys: `cp .env.example .env`, then fill in `ANTHROPIC_API_KEY` and `GITHUB_TOKEN`.
Never commit `.env`.

---

## 9. Desktop dashboard (GUI)

```bash
npm run dashboard
# or
python3 ./ecc_dashboard.py
```

Tkinter app for browsing agents, skills, commands, and rules with search, filtering, and a
dark/light toggle.

**Prerequisites:** Python 3 with Tkinter available (`python3 -c "import tkinter"` must succeed —
on Debian/Ubuntu that's `python3-tk`), and it must be launched from the repo root, since it
imports `scripts/lib/ecc_dashboard_runtime.py` as a namespace package. Tkinter is absent from
plenty of minimal Python builds; this is the most common reason it won't start.

---

## 10. Working on this repo (contributors)

```bash
npm install
node tests/run-all.js
```

`node tests/run-all.js` is the fast inner loop. **It needs `npm install` first** — several
test files exercise `install-apply.js`, which requires `ajv`, and they fail with
`Cannot find module 'ajv'` on a fresh clone rather than skipping.

Single files, when you're iterating on one thing:

```bash
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/hooks/hooks.test.js
```

Full gate — what you run before pushing:

```bash
npm test     # unicode safety + all validators + catalog + registry + full test suite
npm run lint # eslint . && markdownlint '**/*.md' --ignore node_modules
```

> Heads up: `npm test` fails on the current checkout, and not because of your changes.
> `catalog:check` reports the docs claim 79 commands while the tree has 84 (README.md,
> AGENTS.md, the zh-CN docs, `plugin.json`, and `marketplace.json` all disagree with reality),
> and `command-registry:check` reports `docs/COMMAND-REGISTRY.json` is out of date. Fix both
> before assuming your branch broke something.

Regenerate the derived files those two checks guard:

```bash
npm run catalog:sync              # rewrite documented counts to match the tree
npm run command-registry:write    # regenerate docs/COMMAND-REGISTRY.json
```

Individual validators, useful when you've touched one surface:

```bash
node scripts/ci/validate-agents.js
node scripts/ci/validate-skills.js
node scripts/ci/validate-commands.js
node scripts/ci/validate-hooks.js
node scripts/ci/validate-rules.js
node scripts/ci/validate-install-manifests.js
node scripts/ci/validate-no-personal-paths.js
node scripts/ci/check-unicode-safety.js
```

Coverage (80% floor on lines/functions/branches/statements):

```bash
npm run coverage
```

Repo conventions worth knowing before you open a PR: conventional commits; lowercase-hyphen
filenames; new `scripts/lib/` modules need a matching `tests/lib/` test; new hooks need at least
one `tests/hooks/` integration test.

---

## 11. Audits and operator surfaces (advanced)

Harness quality audit — scores hooks, skills, commands, and agents against explicit rules.
Auto-detects whether it's looking at the ECC repo or a consumer project:

```bash
node scripts/harness-audit.js
node scripts/harness-audit.js --scope hooks --format json
```

Operator state, backed by the SQLite state store (needs `npm install` for `sql.js`):

```bash
node scripts/ecc.js status
node scripts/ecc.js status --markdown --write status.md   # portable handoff doc
node scripts/ecc.js status --exit-code                    # non-zero when readiness needs attention
node scripts/ecc.js sessions
node scripts/ecc.js work-items list
node scripts/ecc.js work-items sync-github --repo affaan-m/ECC
```

`status --markdown` is the one to reach for at a handoff: readiness, active sessions, skill-run
health, install health, and linked work items in one file.

Stuck-loop detection — scans Claude transcripts for stale wakeups and pending tool results:

```bash
node scripts/loop-status.js
node scripts/loop-status.js --watch
node scripts/loop-status.js --exit-code   # exit 2 on attention signals
```

Local operator control pane (web UI on localhost):

```bash
npm run control:pane            # defaults to 127.0.0.1:8765, opens a browser
npm run control:pane -- --read-only --no-open
```

Repo/maintainer audits — these read GitHub queues, discussions, roadmap, and release evidence,
so they're maintainer-shaped rather than user-shaped:

```bash
npm run platform:audit
npm run operator:dashboard
npm run observability:ready
```

---

## 12. Multi-agent orchestration (advanced)

```bash
node scripts/orchestrate-worktrees.js <plan.json>             # dry-run plan, prints only
node scripts/orchestrate-worktrees.js <plan.json> --execute   # actually launch
node scripts/orchestration-status.js <session-name|plan.json>
```

Launches one git worktree plus a tmux session per worker from a plan file. Without a flag it only
prints — that's the safe default and how you should look at it first. Requires `tmux` and a plan
JSON; `/orchestrate` (a retired shim, see section 14) documents the plan format.

The `/multi-*` commands (`/multi-plan`, `/multi-execute`, `/multi-backend`, `/multi-frontend`,
`/multi-workflow`) are **not** covered by any ECC install. They need a separate runtime:

```bash
npx ccg-workflow
```

That provides `~/.claude/bin/codeagent-wrapper` and `~/.claude/.ccg/prompts/*`. Without it these
commands will not run correctly.

---

## 13. ECC 2.0 alpha — Rust control plane (advanced, alpha)

```bash
cd ecc2
cargo run -- dashboard
cargo run -- sessions
cargo run -- start --task "audit the repo and propose fixes" --agent claude --worktree
cargo run -- status latest
cargo run -- stop <session-id>
cargo run -- resume <session-id>
cargo run -- daemon
cargo test
```

A terminal UI for managing many agent sessions at once, with a SQLite session store and a
background daemon. Requires a Rust toolchain. The repo's own README for `ecc2/` is explicit that
this is alpha — real code, valid to build and test locally, not a GA release, and missing the
orchestration/diff-review/memory layers that define the finished product. Treat it that way.

---

## 14. Flags — stale, uncertain, or contradictory

Things that will bite you if you trust the docs literally. Each of these was checked against
the tree, not assumed.

- **`COMMANDS-QUICK-REF.md` is stale.** It claims "59 slash commands"; `commands/` holds 84.
  More importantly, ~12 commands it lists are **retired shims** that are no longer part of the
  default command surface: `/tdd`, `/verify`, `/e2e`, `/docs`, `/eval`, `/claw`, `/devfleet`,
  `/orchestrate`, `/context-budget`, `/prompt-optimize`, `/rules-distill`, `/agent-sort`. They now
  live in `legacy-command-shims/commands/` and each one just delegates to a maintained skill
  (`/tdd` → the `tdd-workflow` skill). Prefer the skill; copy the individual shim file into your
  commands directory only if you need the old name.
- **Documented component counts are wrong across the board.** README, AGENTS.md, the zh-CN docs,
  `plugin.json`, and `marketplace.json` all say 79 commands; the tree has 84.
  `docs/COMMAND-REGISTRY.json` is likewise out of date. Both are enforced by `npm test`, so the
  suite is red on a clean checkout until someone runs `npm run catalog:sync` and
  `npm run command-registry:write`. Agent (63) and skill (257) counts do check out.
- **The package-manager env var is inconsistent.** The code reads `CLAUDE_PACKAGE_MANAGER`
  (`scripts/lib/package-manager.js`), but `.env.example` documents `CLAUDE_CODE_PACKAGE_MANAGER`
  and both CI workflows set that second name. Use `CLAUDE_PACKAGE_MANAGER` — that's the one the
  detection code actually checks.
- **`npx ecc ...` is unverified.** README uses `npx ecc consult ...` and `npx ecc install ...`,
  but the published package is `ecc-universal` (its bins are `ecc`, `ecc-install`,
  `ecc-control-pane`). `npx ecc` resolves an npm package named `ecc`, which is not this project.
  The forms verified here are `node scripts/<name>.js ...` from a clone, and `node scripts/ecc.js
  <command>`. If you want the global CLI, `npm i -g ecc-universal` then `ecc <command>` is the
  safer route.
- **`/skill-health` may not work from an npm install.** The command shells out to
  `scripts/skills-health.js`, which is missing from the `files` allowlist in `package.json`, so it
  isn't shipped in the npm tarball. It works fine from a git clone or plugin install.
- **CONTRIBUTING.md points at the old repo name.** Its clone step uses
  `affaan-m/everything-claude-code`; the repo is now `affaan-m/ECC`. Same for the URLs in
  `pyproject.toml`.
- **`commands/security-scan.md` uses the pre-rename plugin namespace** in its frontmatter
  (`agent: everything-claude-code:security-reviewer`); the current plugin identifier is `ecc@ecc`.
  It's the only command file still doing this.
- **`src/llm/` and `pyproject.toml` are an unrelated in-tree Python package** (`llm-abstraction`,
  a provider-agnostic LLM layer with an `llm-select` console script and its own pytest suite under
  `tests/test_*.py`). It isn't mentioned in the README and isn't wired into `npm test`. Inferred
  from the files, not documented — treat as separate from ECC proper.
- **`docs/HERMES-SETUP.md` describes an operator stack you don't have.** It documents the
  `~/.hermes` topology around ECC. It's a reference architecture, not something an ECC install
  gives you.
