# PRD Delivery Orchestration

> **Status: reference specification.** This is not an invocable skill or command.
> It describes a workflow that starts at `/ecc:prp-prd` and drives a PRD to a
> merge-ready PR, with the main session orchestrating and subagents doing the
> work. Several prerequisites are unmet — see [§15 Known blockers](#15-known-blockers).

## Table of contents

1. [Scope and loop unit](#1-scope-and-loop-unit)
2. [Phase 0 preconditions](#2-phase-0-preconditions)
3. [Step chain](#3-step-chain)
4. [State model](#4-state-model)
5. [Plan Adherence Protocol](#5-plan-adherence-protocol)
6. [Boundary set](#6-boundary-set)
7. [Post-lane measurement](#7-post-lane-measurement)
8. [Blind review](#8-blind-review)
9. [Gates](#9-gates)
10. [Status format](#10-status-format)
11. [Failure documentation](#11-failure-documentation)
12. [Terminal states](#12-terminal-states)
13. [Resuming a run](#13-resuming-a-run)
14. [Dispatch templates](#14-dispatch-templates)
15. [Known blockers](#15-known-blockers)

---

## 1. Scope and loop unit

**The loop unit is the phase, not the PRD.** A PRD run is a one-shot; the phase is
what repeats, N times per run and across runs. This distinction matters because a
loop wrapped around a genuinely one-off task is the thing `skills/loop-design-check`
tells you not to build.

### What this workflow composes

It does not reimplement existing ECC assets. It wires them together and adds what
is missing:

| Already exists | Where |
|---|---|
| PRD generation with a phase table | `commands/prp-prd.md` |
| Phase-table iteration (parse, dependency check, select next eligible pending) | `commands/prp-plan.md` lines 18-45 |
| Phase status transitions | `commands/prp-plan.md` (pending→in-progress), `commands/prp-implement.md` (→complete) |
| Gated Research→Plan→TDD→Review→Commit pipeline with GATE 1 / GATE 2 | `skills/orch-pipeline/SKILL.md` |
| Multi-reviewer fan-out with dedup and fail-closed adversarial verification | `workflows/orch-review.workflow.js` |
| Codemap and doc generation | `commands/update-codemaps.md`, `commands/update-docs.md` |
| PR creation | `commands/prp-pr.md` |
| Toolchain detection | `commands/prp-implement.md` lines 18-46 |

**What is genuinely new here:** (a) wiring the PRD phase table into the gated
pipeline — `orch-pipeline` is phase-table-blind and derives its own slices; and
(b) adding codemaps, docs, and PR as pipeline phases, since `orch-pipeline` ends
at Commit.

### Red lines

- The workflow **opens a PR. It never merges, never pushes to a default branch,
  never force-pushes.** Responsibility does not transfer to the machine.
- **Judgment stays with the human.** Execution feedback belongs to the machine;
  every scope decision reaches a person. Per `skills/loop-design-check`.
- **No subagent invokes a slash command.** No agent in this repo has a
  command-invoking tool. Commands run in the main session only.
- **Agent names carry the `ecc:` prefix.** Under a plugin install, bare names force
  fuzzy matching that fails intermittently (`skills/plan-orchestrate/SKILL.md`
  lines 47-57).

### Concurrency rule

**Implementation serializes. Read-only work parallelizes.**

In-session subagents share the main session's working directory and one `.git`
index. There is no per-subagent cwd control, so "each lane works in its own
worktree" reduces to "each lane remembers to `cd`" — a prompt instruction. Prompt
instructions are insufficient here: `~/.claude/skills/commit-before-agent-dispatch`
records a real incident where a documentation subagent ran a repo-wide
`git checkout --`, destroyed five uncommitted edits in another agent's tree, and
then truthfully reported `"Modified: None"`. An explicit "do not touch git state"
instruction in its prompt did not prevent it.

Serialization is not a concession — it is the **enabling condition** for
mechanical enforcement. With exactly one writer in flight, the Gate-1-approved
Files-to-Change set is an unambiguous manifest that a hook can enforce. Under
parallel lanes that manifest is ambiguous and the enforcement is unbuildable.

Parallelize research, review fan-out, and per-file analysis. Those have an empty
write surface and can fan out at any width — and that is where the throughput win
actually was.

> **Note:** `rules/common/agents.md` lines 29-42 says "ALWAYS use parallel Task
> execution for independent operations" with no write-surface caveat. That rule
> outranks the skills carrying the collision guidance. This workflow explicitly
> overrides it for write-capable lanes. See [§15](#15-known-blockers).

Parallel implementation becomes defensible only when **all five** hold — none do
today with in-session dispatch:

1. Each lane is a separate OS process in its own git worktree and branch (via
   `scripts/orchestrate-worktrees.js`), not an in-session subagent.
2. Write surfaces are disjoint **after dependency closure**, scored by
   `scripts/lib/agent-proximity/` — naive path-set intersection misses the
   import-coupling channel, where two lanes touching disjoint files still break
   each other because one imports the other.
3. Shared infra (`package.json`, lockfiles, `pyproject.toml`, test/lint config,
   snapshots, migration ordinals) is an orchestrator-owned lock; a lane needing a
   change there returns `BLOCKED: infra <path>`.
4. Landing goes through a merge queue with eviction, never concurrent `git commit`
   in one index.
5. Every lane's base is a real commit.

---

## 2. Phase 0 preconditions

Run once, before any phase. Reuse `commands/prp-implement.md` lines 18-46 verbatim
for package-manager and script detection — do not reinvent it.

Probe, in order:

| Check | Failure meaning |
|---|---|
| Test runner resolves and executes | ENV — candidate terminal FAILED |
| Typecheck command resolves (or is N/A for the stack) | ENV |
| Lint command resolves | ENV |
| Working tree is clean | Must be resolved before any dispatch |
| **Tests are currently green** | A red baseline makes every later verification meaningless |
| `docs/delivery/LESSONS.md` read in full | Populates the known-hazards index (§11) |

Resolved commands **freeze into `STATE.json.verify_commands`** so they cannot
drift mid-run.

This probe is what makes terminal FAILED reachable rather than theoretical: an
unmet precondition here is precisely the "unfixable environment precondition"
that terminates a run.

> A repo that does not deserve a loop will only have its errors amplified by one.
> If the baseline is red or the toolchain is unresolvable, stop here.

---

## 3. Step chain

```text
/ecc:prp-prd                     main session, interactive, not delegable
  └─> compact (mandatory)        drop research + Q&A transcript
      └─> FOR EACH eligible phase:
            1. plan subagent          writes its own plan file, returns a handoff
            2. GATE 1                 full for phase 1; exception-gated after
            3. implement              2-4 SERIAL slice lanes
            4. review fan-out         parallel, blind
            5. boundary checks        orchestrator-run, 9 rows
            6. GATE 2                 never collapses
            7. commit                 explicit pathspec
            8. codemap refresh        incremental, scoped to touched files
            9. compact (mandatory)    phase boundary
      └─> full codemaps + docs
      └─> PR
      └─> final report
```

### Step details

**Step 0 — PRD.** `/ecc:prp-prd` runs in the main session. It is a three-round
interactive interview with six human stops and dispatches zero subagents; it
cannot be delegated because subagents cannot prompt the user.

**Step 1 — Plan.** One planning subagent per phase. It **writes its own plan file**
and returns only a handoff (≤40 lines: plan path, task titles, file count, risk
flags). The plan text must never transit the orchestrator — see
[§15](#15-known-blockers) for why this requires a new agent.

**Step 3 — Implement.** 2-4 **serial** slice lanes per phase, not one lane per
task. Per-spawn overhead is roughly 4,750-13,000 tokens (agent body, its own file
reads, re-deriving conventions, prompt and return) against 3-15k of productive
work for a small task — 40-70% overhead. Consecutive tasks in a phase usually
touch the same 2-4 files, so per-task lanes perform duplicate reads of the same
files. For a six-task phase: six solo lanes cost ~39k of overhead; three slices
cost ~23k; two slices cost ~17k.

Split a slice when it would exceed ~5 files, cross a module boundary, or contain
an independently-revertible risky change (schema migration, dependency bump). Do
**not** collapse to one lane per phase — that pushes the lane past ~100k of its
own context and loses everything on a mid-way failure with no intermediate commit.

Each lane is preceded and followed by a checkpoint commit (§7), bounding the blast
radius of any clobber to one slice.

**Step 8 — Incremental codemap refresh.** Scoped to files the phase touched.
Without this, phase K+1's planner reads codemaps generated before phase 1 — stale
context that makes a faster loop produce more errors, not fewer. Full regeneration
happens once at the end.

---

## 4. State model

**No shared mutable file has more than one writer, ever. Truth is append-only;
every human-readable artifact is a rendered view.**

| Path | Writer | Mechanism |
|---|---|---|
| `events.jsonl` | main session only | `>>` append, one JSON object per line, <4 KB per record |
| `RUN.md` | main session only | `>>` append |
| `STATE.json` | main session only | atomic write: tmp + rename (pattern at `scripts/lib/session-bridge.js` lines 74-129) |
| `status.md` | main session only | blind `Write` once per phase boundary — **never Read first** |
| PRD phase table | main session only, **only at a gate** | re-rendered from `events.jsonl` |
| `phases/p<N>/…` | the one subagent that owns it | unique path per phase and lane |

**The PRD phase table is a projection, not state.** Re-rendering it from the event
log at gate time dissolves the lost-update race rather than mitigating it: with
one writer and no lane in flight at gate time, read-modify-write is safe, and a
corrupted table is repairable by re-rendering.

`status.md` is capped at ~15 lines and is the **post-compact recovery contract** —
a fresh session resumes from a single `Read` of it. If it is too large to want to
re-read after a compact, it is not doing its job.

Event vocabulary:

```json
{"ts":"2026-08-24T14:03:11Z","phase":2,"task":"t4","actor":"ecc:tdd-guide","event":"lane_return","paths":["src/auth/refresh.ts"],"head":"a1b2c3d","exit_code":0,"evidence":"41/41 green"}
```

`event ∈ {phase_start, plan_ready, gate1_approved, lane_dispatch, lane_return,
verify_pass, verify_fail, invariant_violation, checkpoint_commit, gate2_approved,
phase_commit, blocked, user_answer, phase_done}`

### Paper trail layout

```text
docs/delivery/
  LESSONS.md              STABLE cross-run, append-only, read at every Phase 0   [tracked]
  LESSONS-ARCHIVE.md      pruned entries, never read at Phase 0                  [tracked]
  <prd-slug>/
    RUN.md                narrative log                                          [tracked]
    events.jsonl          append-only truth                                      [tracked]
    MUTATIONS.md          mutation ledger                                        [tracked]
    STATE.json            machine resume point                                   [ignored]
    status.md             projection, derivable                                  [ignored]
    phases/p<N>-<slug>/
      plan.md             FROZEN as dispatched — not the post-mutation version
      verify.txt          raw orchestrator-run command output
      boundary.txt        the 9-row boundary check output
      review/{quality,security,<lang>}.md
      gate1.md gate2.md   what was shown verbatim, and what the human replied
```

Two placement decisions worth stating explicitly:

- **`LESSONS.md` lives at a stable cross-run path**, not inside the per-run
  directory. A lessons file scoped to one run is invisible to the next PRD, so the
  lesson would have zero effect — which is the exact failure the paper trail
  exists to prevent.
- **`RUN.md` and `events.jsonl` are tracked.** They are the forensic record of a
  run that may have destroyed something; a `git checkout` that discards them takes
  the evidence with it. Only genuinely derivable (`status.md`) or transient
  (`STATE.json`) files are ignored.

Suggested `.gitignore` entries:

```gitignore
docs/delivery/*/STATE.json
docs/delivery/*/status.md
```

---

## 5. Plan Adherence Protocol

"No deviations" means **no unilateral scope change by any agent** — not "no
judgment about implementation detail". Only the human may change scope.

This replaces the model in `commands/prp-implement.md` lines 120-126, which grants
an agent authority to diverge and mention it afterward. That is a post-hoc log of
a decision already made, and it is the definition of inventing work.

The text below goes verbatim into every implementer dispatch.

```text
## Plan Adherence Protocol

Two categories. Decide which one you are in before you act.

### A. Adaptation — allowed, no ceremony, do not report it
An implementation detail the plan did not specify. You are filling a
blank, not changing the plan.

Test: after your change, is every line of the plan's task list still
true as written, and is the Files-to-Change set unchanged?
  Yes -> adaptation. Proceed silently.

Examples: naming; splitting a function the plan described as one unit;
adding a test case beyond the ones named; error message wording;
choosing between two equivalent stdlib calls.

### B. Mutation — you have zero authority. Stop.
Anything that changes WHAT is built, WHETHER a task runs, the ORDER of
tasks, or the set of files touched.

Do not do it. Do not do it "and log it." Return BLOCKED and stop.

Triggers: a task's precondition is false; a named file, symbol, or API
does not exist or has a different shape; a task is already done; two
tasks conflict; the fix requires a file not in Files-to-Change; the fix
requires a dependency not named in the plan; the acceptance command
cannot pass as written.

### The one thing you must never do
Never make acceptance pass by changing acceptance. Not the tests, not
the lint config, not the type config, not the coverage threshold, not
the assertion. If the only path to green weakens a check, that is a
BLOCKED result, not a fix.

### BLOCKED return schema
BLOCKED
task:      <task id from plan>
expected:  <what the plan says, quoted>
actual:    <what you found, with file:line evidence>
attempted: <what you tried, N attempts>
options:
  1. <smallest resolving change> - touches <files>
  2. <alternative>               - touches <files>

Then stop. Do not implement any option. Do not ask a question —
you have no user. Return and exit.
```

### Orchestrator-side mutation handling

A subagent never asks and never guesses; it returns `BLOCKED` and exits. Only the
orchestrator, in the main session, asks the human. This is the single rule that
reconciles "intervention is expected" with the failure mode where an agent runs
the wrong answer to completion rather than stopping.

| Class | Definition | Action |
|---|---|---|
| `STALE-REF` | Plan names a path or symbol that moved or was renamed; work identity unchanged | Repair the reference, log MUTATION, redispatch. No gate. |
| `SPLIT` | Task is correct but too large to land atomically | Split into ordered sub-tasks with identical total scope, log MUTATION, redispatch. No gate. |
| `ALREADY-DONE` | Task's postcondition already holds, evidence attached | Mark skipped with evidence, log MUTATION. No gate. |
| `SCOPE` | Resolving needs new files, a new dependency, a new contract, or dropping a deliverable | **ESCALATE to human.** Never auto-resolve. |
| `ENV` | Toolchain, credential, or platform precondition unmet | **ESCALATE to human.** Candidate terminal FAILED. |

The three no-gate classes are all identity-preserving: the set of deliverables is
unchanged. `SCOPE` and `ENV` always reach a person. That is the entirety of
"follow the plan, don't invent work."

**Damping: 3 dispatches per task, 2 orchestrator mutations per phase.** Exceeding
either transitions to SUSPENDED (§12).

`MUTATIONS.md` row format:

```text
| ts | phase | task | class | from | to | authorized_by |
```

where `authorized_by` is `orchestrator` or `human@gate<N>`.

---

## 6. Boundary set

Run by the **orchestrator** after every phase, before Gate 2. Deterministic and
phase-scoped. Any non-zero is a hard block, not an advisory.

```text
BOUNDARY CHECKS
1. test files deleted ..................... must be 0
2. assertions removed (net) ............... must be <= 0
3. skip/xfail/.only/todo added ............ must be 0
4. coverage threshold lowered ............. must be 0
5. lint rules disabled / files ignored .... must be 0
6. type strictness lowered ................ must be 0
7. files touched outside Files-to-Change .. must be 0
8. dependencies added not in the plan ..... must be 0
9. acceptance commands edited ............. must be 0
```

Rows 1-6 exist because a done-criterion without a boundary set invites the
degenerate solution: an agent gated only on "tests pass" can make tests pass by
weakening tests.

**Rows 7 and 8 are "don't invent work" in a form a machine can decide.** This is
why the boundary set — not a deviation policy — is the correct home for that
constraint.

Output goes to `phases/p<N>/boundary.txt` and is summarized in the Gate 2 block.

---

## 7. Post-lane measurement

**Agent self-reports are evidence of belief, not fact.** In the documented
incident, the report `"Modified: None (no changes outside CODEMAPS/)"` was
literally true and completely misleading.

Before dispatching any write-capable lane, assert `git status --porcelain` is
empty; commit if not. This promotes `~/.claude/skills/commit-before-agent-dispatch`
from advice to a hard precondition, and it is the **only** mechanism that converts
an unrecoverable loss into a recoverable one.

After every lane returns, before anything else, run and record to `events.jsonl`:

1. **`git status --porcelain`** — every path must be in the lane's allowed set.
2. **`git log --oneline -3`** against pre-dispatch HEAD, plus **`git stash list`** —
   a lane must not have committed or stashed anything.
3. **Disappearance check.** Diff the pre-dispatch `git status --porcelain` against
   the post-dispatch one. Any path that was modified before and is unmodified now
   was **reverted**. This is the peer-clobber signature and the only reliable
   detector: `git checkout --` on unstaged content leaves no reflog entry (HEAD
   does not move), an empty stash list, a clean `git status`, and nothing for
   `git fsck --lost-found` to find because the content was never staged. **Do not
   waste a step on `fsck`.**
4. **`stat -c '%y %n'`** over the lane's declared files — two or more unrelated
   files with mtimes inside the same millisecond means one git operation, not two
   edits.
5. **`git diff --stat HEAD`** — a lane reporting success with an empty diff is a
   **failure**, not a success.
6. **Re-run the frozen verify commands and read the exit code.** Never accept
   "tests pass" from a report.
7. **Named-test presence.** Gate 1 recorded a specific test name per task; assert
   `grep -rn "<test name>"` finds it *and* that it appears in the runner output.
   Check for the artifact; do not ask whether it was produced. Per
   `skills/blind-independent-verification`.

Any of 1-4 failing is an `invariant_violation`: **hard stop, restore from the
checkpoint commit, surface to the user.** It is not a `BLOCKED` to be folded in
and resumed. Asking the user for permission to continue after a clobber compounds
the loss with the user's own consent as cover.

### Residual risks, stated plainly

- A Bash-capable agent can write via `python -c`, `tee`, or a heredoc, which a
  `Write`/`Edit` path guard never sees. Checkpoint commits and the measurement
  above are what cover that, not the guard.
- `scripts/hooks/gateguard-fact-force.js` detects destructive git but **must not
  be counted as mitigation here** — see [§15](#15-known-blockers).

---

## 8. Blind review

Reviewers receive:

- `git diff <phase-base>..HEAD`
- the **frozen** `phases/p<N>/plan.md`
- the phase acceptance criteria

Reviewers are **denied**: the implementer's report, its self-assessment, and its
deviation notes. A judge handed the defendant's framing inherits it. Per
`skills/blind-independent-verification`.

Fan-out is parallel and delegates to `workflows/orch-review.workflow.js`, which
already implements one reviewer per dimension, dedup on normalized evidence
(11 raw findings → 4 unique in its own testing), and adversarial verification that
**refuses to refute on uncertainty** so an unverifiable CRITICAL is never demoted
to advisory. It fails closed: a dead reviewer means the verdict can never be a
clean `APPROVE`, and a dead verifier leaves the finding blocking and tagged
unverified.

Do not hand-roll this fan-out. Note the contract mismatch in
[§15](#15-known-blockers) before wiring it.

The language reviewer must be **computed, not assumed**: this repo is polyglot
(Python and JavaScript). Where no language exceeds ~60% of the diff, fall back to
plain `ecc:code-reviewer`.

---

## 9. Gates

Two gates per phase satisfies the letter of "judgment stays with the human," but
2×N approvals in one sitting will degrade into rubber-stamping — and **a
rubber-stamped gate is worse than no gate: it manufactures assurance without
supplying judgment.**

### GATE 1 — after plan, before the first write dispatch

Full for phase 1. Thereafter it **collapses to a 5-line notify-only checkpoint**
unless the phase introduces any of:

- a new dependency
- a new public contract or API
- a file outside the PRD's declared scope
- a security trigger (per `skills/orch-pipeline/SKILL.md` lines 100-104)
- a task count exceeding 150% of the previous phase

Any one of those restores the full gate. The reasoning: the human already approved
the phase table in the PRD, so a per-phase plan is a mechanical expansion of an
approved phase. Spend attention where the blast radius actually changed.

What the human approves is not prose — it is the **lane manifest**: per task, the
allowed-paths set, infra files claimed, the verify command, and the named test
that will prove it. Approval is the act that populates the allowed-path manifest.
*A gate whose output does not become machine-enforced configuration is decoration.*

### GATE 2 — after review and boundary checks, before commit

**Never collapses.** It is the durable-state boundary and the last point at which
responsibility can stay with a person.

Presented **by reference**: `git diff --stat`, the proposed commit message, and
the `blocking[]` findings — never the full diff text.

### Anti-rubber-stamp measures

- **Never present a recommendation.** "All green — approve?" is a leading question.
- **`Look here first` is mandatory and never empty.** If boundary and verify checks
  are all clean, the orchestrator must still name the two changes most likely to be
  wrong: the largest hunk, the most-coupled file, anything flagged advisory. A gate
  answerable without reading anything is not a gate.
- **Every Gate 2 carries one judgment question only the human can answer** — surface
  the PRD hypothesis and ask whether this phase moved it. The question changes every
  phase and cannot be answered by reflex.
- **Approval requires a non-trivial token** — the phase number, not "y".
- **Budget the human.** After 3 completed phases or ~90 minutes, write `STATE.json`
  and offer to resume in a fresh session.
- **Blanket pre-approval is allowed for Gate 1 only and refused for Gate 2**, with
  the reason stated: pre-approving Gate 2 removes the high-level feedback loop and
  the run degrades to unattended. In that mode the workflow still stops before
  opening a PR.

> A third, undeclared gate exists in `commands/update-codemaps.md` line 51 — it
> prompts for approval when the codemap diff exceeds 30%. Fold it into the final
> report rather than letting it ambush the human mid-run.

---

## 10. Status format

Two tiers. A fixed-width block at every step would defeat the leanness goal; a
single line per step would be too thin at a decision point.

### Per step — one line, ~40 tokens

```text
[P<phase>/<N> S<step>/<M>] <step-name> — <STATE> <elapsed>  <one fact>
```

`STATE ∈ RUN | OK | RETRY n/3 | BLOCKED | SKIP`

```text
[P2/5 S1/6] plan                 — OK 1m48s  9 tasks, 4 files, 2 slices
[P2/5 S2/6] gate-1               — OK        collapsed: no new deps/contracts
[P2/5 S3/6] slice:token-refresh  — OK 3m02s  red->green, +2 tests
[P2/5 S4/6] slice:session-store  — RETRY 1/3 typecheck: 2 errors
[P2/5 S4/6] slice:session-store  — OK 1m11s  red->green, +1 test
[P2/5 S5/6] review x3 (parallel) — OK 2m10s  0 CRITICAL, 1 HIGH, 3 advisory
[P2/5 S6/6] gate-2               — RUN
```

### At gates only — fixed-width block, ~20 lines

```text
PHASE 2/5 — Token refresh                          GATE 2 · COMMIT
==================================================================
Verify    tests 41/41   types 0   lint 0   build ok   cov 84% (+2)
Boundary  tests deleted 0 | skips added 0 | thresholds lowered 0
          files outside plan 0 | new deps 0 | acceptance edited 0
Diff      6 files  +182 -34   (src/auth/*, tests/auth/*)
Mutations 1 STALE-REF (task 4: authStore -> sessionStore, orchestrator)
Blockers  0 open · 1 resolved -> LESSONS L-0007 (draft below)
Reviews   0 CRITICAL  1 HIGH (resolved)  3 advisory (deferred)

Look here first:
  1. src/auth/refresh.ts:88 — retry backoff is a fixed 200ms, not the
     exponential the PRD latency budget assumes.
  2. 3 advisory findings deferred — RUN.md#p2-review.

Judgment call (only you can answer):
  PRD hypothesis: "users stop re-authing mid-session."
  This phase refreshes tokens silently but does NOT extend idle
  sessions. Does that still move the hypothesis?

Commit as: feat(auth): silent token refresh with backoff retry
Reply "2" to approve, or state the change.
==================================================================
```

### Subagent return contract

```text
Return AT MOST 15 lines. Fixed keys, no prose preamble, no restatement
of the task. Status is one of: OK | BLOCKED | FAIL.

status:   OK
tasks:    4/4
files:    src/auth/refresh.ts (+88), tests/auth/refresh.test.ts (+41)
tests:    +3 written, 41/41 green
verify:   <exact commands run, one per line>
adapted:  <none | one line each, max 3>
notes:    <max 2 lines, only if the orchestrator must act on it>

Do not include: reasoning, a self-assessment of quality, a summary of
the plan, or "I have successfully...". The orchestrator re-runs verify
itself; your claim is not evidence.
```

---

## 11. Failure documentation

### When an entry is written

On **every resolved blocker that meets the threshold** — not only on terminal
failure. A terminal-only policy produces a file that is empty in the overwhelming
majority of runs and never earns its keep; the knowledge worth capturing is
generated by blockers that *were* fixed.

> Write an entry if the blocker cost **≥2 fix attempts**, OR forced a **plan
> mutation of any class**, OR was an **environment/tooling defect**, OR a reviewer
> raised a **CRITICAL/HIGH**. Otherwise skip. Do not write an entry for a one-shot
> typo fix.

Entries are also written on SUSPENDED and FAILED.

Each entry is shown at the **Gate 2 of the phase that produced it** and is
human-approved before becoming durable. This workflow rewrites its own operating
rules, and self-rewriting rules need *stricter* human review, not less.

### Entry format

```markdown
### L-0007 · Vitest globals off breaks every new test file
<!-- hits: 2 | seen: prd-auth#p2, prd-billing#p1 | last: 2026-08-24 | promoted: yes -->

**Signal:** a brand-new test file fails with `describe is not defined`
while every existing test file passes.

**Root cause:** `vitest.config.ts` sets `globals: false`; existing test
files import from `vitest` explicitly. Files written from memory omit
the import, so the failure looks like a runner problem, not an import
problem.

**Next time:** before writing the first test of a phase, grep `globals:`
in the vitest/jest config; if false or absent, the implementer brief
must carry the literal line
`import { describe, it, expect } from 'vitest'`.

**Boundary promoted:** yes — implementer brief template now carries the
import line for this repo. (promoted at hits=2)

**Cost:** 2 retries, ~6 min.
```

Field rules, mapped to `skills/growth-log`:

- The **title names the pattern, not the event** (failures over achievements).
- **`hits` plus the merge procedure below** (search before writing; same root
  cause means merge, never duplicate).
- **`Next time` is the transferability sentence.** An entry without one is
  rejected.
- `Cost` exists to justify pruning. `promoted` is the field that gives the lesson
  teeth.

**Merge procedure.** Before writing, grep `LESSONS.md` for the root-cause
keywords. On a match — same root cause, different symptom — increment `hits`,
append the phase to `seen:`, update `last:`, and add the new symptom as a bullet
under **Signal**. Do not create a second entry. The entry title is the root cause;
symptoms are bullets under it.

### How a later run consumes it

In ascending order of reliability:

1. **Mandatory Phase 0 read.** The orchestrator reads `docs/delivery/LESSONS.md`
   in full and builds a known-hazards index keyed on `Signal:` tokens.
2. **Per-phase injection.** Match entries whose Signal mentions the phase's files,
   directories, commands, or stack tokens. Paste the matched **Next time**
   sentences verbatim into the planner and implementer briefs under a
   `## Known hazards` heading, capped at 5 per brief, highest `hits` first.
3. **Promotion at `hits >= 2`.** The lesson MUST be converted into one of: a row in
   the boundary set (§6), a Phase 0 precondition probe (§2), or a permanent line in
   `CLAUDE.md` / `rules/`. Then set `promoted: yes`.

**A lesson that is read but never promoted has no effect.** Reading is advisory
and will eventually be ignored; promotion is what turns "we learned not to do
that" into something that cannot recur.

**Pruning.** At the final report, entries with `hits: 1` and `last:` older than 180
days move to `LESSONS-ARCHIVE.md`. This keeps the always-read file bounded, which
is what makes mechanism 1 affordable.

### Relationship to the instinct system

`LESSONS.md` is primary. Instincts under `skills/continuous-learning-v2` live at
`${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/` — **outside the repo** — so they
are invisible in code review, do not travel with a clone, and cannot serve as an
in-repo delivery paper trail.

Secondary and opt-in, offered at the final report only, never auto-run:

```text
Offer /ecc:learn-eval over the run's LESSONS.md delta.
  repo-specific hazard -> stays in docs/delivery/LESSONS.md. No instinct.
  cross-project hazard -> candidate for a global skill/instinct via
                          learn-eval's quality gate.

Never auto-save an instinct.
```

---

## 12. Terminal states

| State | Terminal | Trigger | Action |
|---|---|---|---|
| **COMPLETE** | yes | All phases done, docs generated, PR open | Final report |
| **FAILED** | yes | User explicitly stops, or an unfixable environment precondition (§2) | Full incident report |
| **SUSPENDED** | **no** | Retry cap hit (3 dispatches/task or 2 mutations/phase), or an unresolved `SCOPE` blocker | Commit or branch-stash all work, write `RUN.md` + `STATE.json` + lessons, emit the status block, halt for a human decision |

SUSPENDED is what gives the loop a floor. Without it, a phase that fails
verification indefinitely has no defined halt — it retries or waits forever, which
is the undamped-loop failure. SUSPENDED honors "failing is not cause to abandon
the workflow" while bounding it: the run is resumable (§13), not abandoned.

### FAILED incident report

```text
## Workflow FAILED: <prd-slug>

Stopped at: Phase <n> / Step <step-name>
Exact state: branch <branch>, last good commit <sha>, files touched: <list>

What was attempted:
  <step-by-step account before failure>

What failed and why:
  <full error / validation output — NOT truncated; this is the evidence>

Blockers hit and resolved earlier in this run (<count>):
  <each: symptom -> resolution>

Root cause:
  <best-known explanation>

LESSONS FOR NEXT ATTEMPT (also written to docs/delivery/LESSONS.md):
  - <transferable rule 1>
  - <transferable rule 2>

Resume: docs/delivery/LESSONS.md loads automatically at the next Phase 0.
Artifacts: RUN.md, events.jsonl, MUTATIONS.md in docs/delivery/<slug>/
```

### COMPLETE report

Phase table, docs/codemap commit, PR URL and CI status, run-log path — plus any
`LESSONS.md` entries accumulated, since a successful run still generates useful
pitfalls.

---

## 13. Resuming a run

1. Read `docs/delivery/<prd-slug>/STATE.json`.
2. Verify `last_commit == git rev-parse HEAD`. On mismatch, stop and report — the
   tree moved since the run halted.
3. Re-emit the last status line.
4. Continue from `phase` / `step` / `task`.

`STATE.json` is required because the PRD's phase-granular `Status` column
(`pending | in-progress | complete`) cannot express "phase 3, task 4 of 9, Gate 1
passed, one `STALE-REF` mutation applied, boundary check pending." A session that
dies mid-phase without it loses everything.

Minimum contents:

```json
{
  "prd_path": "…", "branch": "…", "phase": 3, "step": 4, "task": "…",
  "gates_passed": ["p1g1", "p1g2", "p2g1"], "mutations": [],
  "open_blockers": [], "retry_counts": {}, "last_commit": "…",
  "verify_commands": {}, "boundary_set": [], "injected_lessons": []
}
```

---

## 14. Dispatch templates

Without literal templates this spec fails the cold-start test — "plan (subagent)"
is not an executable instruction. A fresh agent must be able to construct a
dispatch by filling slots, not by inventing one.

### Planner

```text
agentType: ecc:<write-capable planner>   # see §15 — does not exist yet
model:     sonnet

REQUIRED INPUTS
  - PRD path and the selected phase row (verbatim)
  - phase Goal / Scope / Verify commands
  - repo conventions from CLAUDE.md and rules/
  - ## Known hazards  (matched LESSONS entries, max 5)

FORBIDDEN INPUTS
  - prior phases' implementer reports

TASK
  Produce a plan following commands/prp-plan.md's template. Write it to
  .claude/PRPs/plans/<phase-slug>.plan.md yourself. Every task must carry
  ACTION / IMPLEMENT / MIRROR / IMPORTS / GOTCHA / VALIDATE, and the plan
  must pass the No Prior Knowledge Test: a developer unfamiliar with this
  codebase implements from the plan alone, without searching or asking.

  Group tasks into 2-4 slices. Each slice: <=5 files, one module, one
  red->green->refactor cycle. Name the specific test that proves each task.

  If the phase is ambiguous, do NOT guess. Return BLOCKED per the
  Ambiguity Gate (commands/prp-plan.md lines 80-89).

RETURN (<=40 lines)
  plan_path: <path>
  slices:    <n>  (title, file count, named test each)
  files:     <total>
  risks:     <max 3 lines>
  Do NOT return the plan text.
```

### Implementer

```text
agentType: ecc:tdd-guide
model:     sonnet
dispatch:  ONE AT A TIME. Never in parallel with another write-capable lane.

REQUIRED INPUTS
  - the frozen plan slice (tasks, Files-to-Change, named tests)
  - allowed-paths manifest approved at GATE 1
  - frozen verify commands from STATE.json
  - ## Known hazards  (matched LESSONS entries, max 5)
  - [Plan Adherence Protocol — §5, verbatim]
  - [return contract — §10, verbatim]

FORBIDDEN
  - touching any path outside the allowed-paths manifest
  - git commit, stash, checkout, restore, reset, clean, rebase, merge, push
  - modifying tests, lint config, type config, or coverage thresholds
    to make acceptance pass

TASK
  For each task in the slice, in the order given: write the failing test
  first, implement minimally to green, refactor. Run the frozen verify
  commands after each task.
```

### Reviewer

```text
Delegate to workflows/orch-review.workflow.js — see §15 for the contract
mismatch that must be resolved first.

REQUIRED INPUTS
  - git diff <phase-base>..HEAD
  - the FROZEN phases/p<N>/plan.md
  - phase acceptance criteria
  - changedFiles[]
  - language (computed; fall back to plain code-reviewer if polyglot)

FORBIDDEN INPUTS
  - the implementer's report, self-assessment, or deviation notes

The workflow supplies dimension fan-out, evidence dedup, and fail-closed
adversarial verification. Do not hand-roll these.
```

---

## 15. Known blockers

Each item below is a verified defect or gap in an existing file that this spec
depends on. **They are documented here, not fixed.** Line numbers were checked
against the working tree as of this document's authoring.

### Blocks the leanness goal

**`workflows/orch-review.workflow.js` requires the full diff as a string.**
Lines 161-162 throw when `args.diff` is missing or empty, and the caller contract
at line 18 marks it required; `workflows/README.md` states the main loop computes
it. This forces the entire phase diff (500-2,000 lines, roughly 5-20k tokens)
through the orchestrator **twice** — once to materialize it, once to pass it as a
tool argument — at 10-40k tokens per phase. A `diffPath` contract, where the caller
writes `git diff` to a temp file and the script reads it, would reduce the
orchestrator's cost to a path string. This is the single defect that most directly
defeats the workflow's stated leanness goal, and §8 cannot be wired as specified
until it is resolved.

**`agents/planner.md` is read-only.** Its frontmatter is `tools: Read, Grep, Glob`
with `model: opus`. It cannot write its plan to disk, so a 400-800 line plan must
return through the orchestrator's context *and* be re-emitted as a `Write`
argument — roughly 50-90k tokens across a five-phase run, for content the
orchestrator never reads again. "Lean orchestrator" and "read-only planner" are
mutually exclusive; there is no third option. **The fix is a new sibling agent, not
adding `Write` to `planner`** — `planner` is shared and used read-only elsewhere.
Two in-repo precedents exist: `agents/gan-planner.md` (sonnet, `Read, Write, Grep,
Glob` — already this shape) and `agents/spec-miner.md`, which establishes the
frontmatter guardrail pattern for a scoped writer. Until this exists, §14's planner
template has no agent to target.

### Blocks machine-decidable completion

**The PRD phase table has no verify commands.** `commands/prp-prd.md` lines 313 and
318 give each phase a free-text `**Success signal**: {How we know it's done}`.
Prose is not a comparator. Without a `Verify` column holding literal shell commands
whose exit codes decide the phase, phase completion is not machine-decidable and
the loop has no automatable done-criterion — which is a veto condition under
`skills/loop-design-check`.

### Safety

**GateGuard must not be counted as clobber mitigation.**
`scripts/hooks/gateguard-fact-force.js` detects destructive git at line 447
(`isDestructiveGit`, reached from line 1245 via `isDestructiveBash`), covering
`git checkout --`, `reset --hard`, `clean -f`, and force-push. But enforcement is
**deny-once-then-allow**: lines 1248-1249 check `isChecked(key)` and
`markChecked(key)`, so an agent that reissues the identical command proceeds. The
checked-key store is session-scoped and shared across subagents, meaning one lane
burning the first attempt **primes the allow for another lane**. Under fan-out it
is worse than no guard, because it reads as protection while providing none. This
is why §1 serializes and §7 relies on checkpoint commits rather than the hook.

**`rules/common/agents.md` lines 29-42 pushes toward unsafe fan-out.** It states
"ALWAYS use parallel Task execution for independent operations" with no
write-surface caveat. It is a *rule*, so it outranks the skills that carry the
collision guidance (`skills/parallel-execution-optimizer`,
`skills/team-agent-orchestration`), and it will actively push an orchestrator
toward exactly the parallel write dispatch §1 forbids. That section already grew a
"Delegation Completion Contract" after an observed zombie-task incident; it needs
the same treatment for write surfaces.

**`commands/prp-commit.md` line 32 stages the whole tree.** Blank arguments map to
`git add -A`, which sweeps anything else present into the commit. This workflow
must always pass an explicit pathspec derived from the lane manifest and verify
with `git diff --cached --stat` before committing.

### Correctness and completeness

**`agents/doc-updater.md` line 3 is factually wrong.** Its description claims it
"Runs `/update-codemaps` and `/update-docs`", but its tools are
`Read, Write, Edit, Bash, Grep, Glob` — no command-invoking tool exists in this
repo. Its body correctly runs generators directly. `docs/COMMAND-AGENT-MAP.md`
records the true direction (`/update-codemaps` → `doc-updater`). The description
will mislead anyone composing agents.

**The adversarial verifier is unrouted.** In `workflows/orch-review.workflow.js`,
the reviewer dimensions set `agentType` explicitly (lines 182-184:
`ecc:code-reviewer`, the language reviewer, and `ecc:security-reviewer`), but the
verify call at line 245 passes only `phase`, `label`, and `schema` — no
`agentType`, so the verifier inherits the default. The verifier decides whether a
CRITICAL ships and runs on only ~4 findings after dedup, making it the cheapest
place in the pipeline to buy a stronger model.

**Two reviewers define no output schema.** `agents/security-reviewer.md` and
`agents/typescript-reviewer.md` specify approval criteria but no output format.
Gate 2 (§9) is specified to present evidence, and it cannot be evidence-based if
its inputs are free prose. Additionally, `typescript-reviewer` will refuse to
review while required CI checks are failing or pending — the workflow needs an
intervention path for that.

**`/orch-review` is not installed.** `skills/orch-review/` does not exist as a
skill directory; the real assets are `commands/orch-review.md` and
`workflows/orch-review.workflow.js`, and `workflows/README.md` lists installer
wiring and the `orch-pipeline` Review-phase integration as unshipped follow-ups.
The spec cannot assume the command is available.

**`commands/update-codemaps.md` line 51 is an undeclared third gate.** It requests
user approval when the codemap diff exceeds 30%. An unbudgeted interrupt is worse
than a budgeted one; fold it into the final report.

### Model routing

Not a defect, but a recommendation the spec assumes: run the **orchestrator on
sonnet** (dispatch, table parsing, and presentation are mechanical, and running
opus across a long conversation is the largest single line item), keep the
**per-phase planner on sonnet** (ambiguity was resolved during the PRD interview;
expanding a decided phase is implementation planning, and opus is justified once
at architecture time rather than N times), and route **opus to the adversarial
verifier** above. Per `commands/model-route.md`.
