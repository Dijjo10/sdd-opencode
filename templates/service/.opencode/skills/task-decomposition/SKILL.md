---
name: task-decomposition
description: Methodology for breaking macro-tasks into atomic micro-tasks and for keeping the task files small. Use this skill when a macro-task is ready to be broken down (its ledger exists with an approved Plan section but no Micro-tasks section yet), when revising a decomposition, when creating or updating tasks.md / task-log.md, and at wrap-up for the compaction step. Defines task IDs and types, the version folder layout, the ledger + execution log pair, and the size budgets.
---

# Task-Decomposition — From Macro to Micro

You are in **task-decomposition mode**. Your job is to decompose a macro-
task into atomic micro-tasks and to maintain the task documentation
files. Every macro-task has **two files**, plus two service-wide indexes:

- `.docs/tasks/tasks.md` — index of macro-tasks, one line each
- `.docs/tasks/task-log.md` — chronological timeline, one line per event
- `<id>.md` — the **ledger**: Summary, Plan, Micro-tasks, Emergent
  decisions, Wrap-up. Small, read often.
- `<id>.log.md` — the **execution log**: completion logs and review
  outputs, append-only. Large, read rarely.

Decomposition operates on the macro-task ledger: it reads the Plan
section (already written) and produces the Micro-tasks section.

## Operating principles

1. **Start from the plan**. Every macro-task has an approved Plan
   section in its ledger (`<id>.md`), written by the
   architect-service skill in macro-task planning mode. Read it first
   — it contains the objective, scope, architecture, design decisions,
   and approach. Decomposition realizes that plan step by step. Do not
   re-open architectural decisions already made in the Plan during
   decomposition; if something in the plan turns out to be wrong, flag
   it and ask the user whether to revise the Plan before continuing.
2. **Atomic means atomic**. A micro-task is implementable in a single
   session, testable in isolation, and delivers a concrete piece of
   value. If it needs "and then also…", split it.
3. **Acceptance criteria are measurable**. "Works correctly" is not a
   criterion. "Returns 200 with JSON body matching schema X for valid
   input" is.
4. **Flag security-relevance liberally**. Err on the side of flagging.
   A false-positive costs one extra review; a false-negative costs a
   production incident.
5. **Don't over-plan**. Detailed implementation plans for micro-tasks
   are produced just-in-time, in chat, when the micro-task is about to
   start. The Micro-tasks section entries in the ledger are
   deliberately high-level.
6. **Separate intent from discovery**. Architectural and design
   choices are already captured in the Plan section of the ledger and
   are the **intent** of the macro-task. The Emergent decisions
   section serves a different purpose: it is a **log of discoveries
   and emergent decisions** that happen *after* planning — during
   detailing or implementation, when reality surprises the plan.
   Examples of what goes into Emergent decisions:
   - A library chosen in the Plan turns out to lack feature X; a
     workaround is adopted
   - During implementation, a micro-task scope is revised because
     it turned out more complex than decomposition estimated
   - A tactical staging choice emerges during detailing that wasn't
     obvious at decomposition time
   Decomposition itself rarely produces emergent decisions — if you
   find yourself making significant architectural choices during
   decomposition, they probably belong back in the Plan (propose
   revising the plan rather than recording as emergent).
7. **Replacement includes cleanup**. If a micro-task replaces existing
   behavior (refactor, rename, swap implementation, migrate from
   pattern A to pattern B), the cleanup of what's being replaced is
   part of the **same** micro-task — not a separate one. State this
   explicitly in the acceptance criteria. Avoid creating "implement
   new X" + "remove old X" as two separate micro-tasks unless the
   migration genuinely needs to be staged across releases (in which
   case document the staging plan in the Plan section of the ledger,
   under Approach).
8. **Tests live with the code**. Implementation and its tests are one
   micro-task. Never decompose "implement X" and "test X" into two
   micro-tasks: the acceptance gate must be real, not postponed.
9. **Unverified assumptions get a risk note**. When a micro-task relies
   on something not yet verified ("the CLI supports this flag", "the
   library exposes this API"), write a **Risk note** with an explicit
   plan B. The builder then never stalls on a wrong assumption.

## Atomicity criteria

A micro-task passes atomicity if **all** of these hold:

- Can be implemented in one focused session (rough target: 1–4 hours of
  agent + human time)
- Has a well-defined "done" state (acceptance criteria you can check)
- Has its own tests (not necessarily a new test file — new test cases
  are fine)
- Does not require completing another pending micro-task to be valuable
- Produces a commit-worthy change (even if the final deliverable spans
  many commits)

If you're tempted to write "implement the whole module", stop — that's
a macro-task. Break it down.

## Security-relevant flag — when to set true

Flag `security-relevant: true` when the micro-task involves any of:

- Authentication (login, session, token handling)
- Authorization (role checks, permission enforcement, access control)
- Cryptography (signing, encryption, hashing, key management)
- Input validation on data from untrusted sources (HTTP bodies, query
  params, webhook payloads, event payloads from external producers)
- Secret handling (reading env secrets, accessing vaults, injecting
  credentials)
- Personally identifiable information (storing, transmitting, logging)
- Network egress to external services (especially with credentials)
- Audit logging of security-relevant events
- Dependency updates affecting crypto / auth / networking libraries

When in doubt, flag true. The cost is one security-review pass on code
that didn't need it.

## State model

Both macro-tasks (in `tasks.md`) and micro-tasks (in the Micro-tasks
section of the macro-task ledger) use the same state values:

| State         | Meaning                                                         |
| ------------- | --------------------------------------------------------------- |
| `pending`         | Defined but not started                                              |
| `in-progress`     | Session active or interrupted mid-work                               |
| `blocked`         | Cannot progress due to external dependency — requires `reason`       |
| `skipped`         | Deliberately not done — requires `reason` and resolution             |
| `wrap-up-pending` | Macro-task only: all micro-tasks `done`, awaiting planner wrap-up    |
| `done`            | Completed and review passed (macro-task: also wrap-up performed)     |

`blocked` and `skipped` **must** include a reason field explaining the
blocker or the decision rationale. Never leave a task in these states
without a note.

`wrap-up-pending` applies **only to macro-tasks** and exists to make
the wrap-up trigger deterministic. When the builder completes the last
micro-task of a macro, it sets the macro-task status to
`wrap-up-pending`. The next session, the planner detects this state
during triage and enters wrap-up mode immediately. Once the planner
finishes the wrap-up cycle, the macro-task transitions to `done`.

## Task IDs, types and layout

### Macro-task ID

```
<NN>[<letter>]-<type>-<slug>        e.g. 03-feat-auth, 12b-fix-token-refresh
```

- **`NN`** — two digits, sequential **per service**, never reused and
  never renumbered (a stable identifier, not an execution order; the
  real order lives in `task-log.md`). Numbering does not restart per
  version, so a macro-task can move to another version without
  changing its ID.
- **`<letter>`** (optional) — a follow-up or insertion tied to an
  existing task: `12b` hardens or completes `12`. Use it only when the
  link to the parent matters; otherwise take the next `NN`.
- **`<type>`** — one of:

  | Type | Use for | Commit type |
  | ---- | ------- | ----------- |
  | `feat` | new behavior | `feat` |
  | `fix` | a bug that needs planning (trivial fixes go to express mode) | `fix` |
  | `ref` | refactor created to change direction (see below) | `refactor` |
  | `chore` | tooling, dependencies, infra, docs-only work | `chore` / `build` / `docs` |
  | `spike` | time-boxed investigation; output is knowledge, not shippable code | `chore` |

- **`<slug>`** — 2-4 lowercase words, dash-separated.

The type also names the branch (`git-workflow` skill):
`<type>/[v<x.y>-]<NN[letter]>-<slug>`, e.g. `feat/v0.2-03-auth`.

**`ref` macro-tasks** have special semantics: they invalidate or
supersede earlier decisions, often introduce `SDD-KEEP` markers for
staged migrations, and their Plan section is the source of truth for the
target architecture until the wrap-up updates `specs.md` /
`architecture.md` / `modules/*.md`. Use `ref` only for work created
specifically to change direction — not for any task that happens to
touch existing code.

### Micro-task ID

`mt-NN` inside its macro-task (`mt-01`, `mt-02`). A micro-task split or
inserted later takes a letter: `mt-03a`, `mt-03b`. Never renumber.
References from outside the ledger use `<macro-id>/mt-NN`
(e.g. `03-feat-auth/mt-02`).

### Folder layout

```
.docs/tasks/
├── tasks.md                 # index, one line per macro-task
├── task-log.md              # timeline, one line per event
├── v0.2/                    # only when versioning.enabled: true
│   ├── 03-feat-auth.md      # ledger
│   ├── 03-feat-auth.log.md  # execution log
│   └── 04-ref-persistence.md
└── backlog/                 # macro-tasks not yet assigned to a version
```

- `versioning.enabled: true` (in `../sdd.config.yaml`): each macro-task
  lives in the folder of the version it ships in, `v<major>.<minor>/`.
  Unscheduled ideas live in `backlog/`. Moving a macro-task to another
  version = moving its two files; the ID does not change.
- `versioning.enabled: false` or no config: the files live directly in
  `.docs/tasks/`.

## Output artifacts

### Size budgets

The task files are read by agents in every session. Their size is a
direct cost, and it grows with every macro-task. These budgets are part
of the format, not a suggestion:

| File | Budget | When it is read |
| ---- | ------ | --------------- |
| `tasks.md` | one line per macro-task | triage, every session |
| `task-log.md` | one line (≤ 200 chars) per event | rarely |
| `<id>.md` (ledger) | Plan 100-300 lines; ED entries ≤ 10 lines each; Summary ≤ 30 lines | detailing, wrap-up |
| `<id>.log.md` | completion log ≤ 15 lines; review verdict 1 line + findings | wrap-up only |

Narrative belongs in chat, not in these files. If a ledger keeps
growing past its budget, the macro-task is probably too big: split it.

### `.docs/tasks/tasks.md` — template

````markdown
# Tasks — <service-name>

> Macro-task index, one line each. States: pending / in-progress /
> wrap-up-pending / done / blocked / skipped.

## Active

- [03-feat-auth](v0.2/03-feat-auth.md) — in-progress — <objective> — branch `feat/v0.2-03-auth`
- [04-ref-persistence](v0.2/04-ref-persistence.md) — pending — <objective> — after 03

## Blocked

- [NN-<type>-<slug>](<path>) — blocked — <reason, who unblocks>

## Skipped

- [NN-<type>-<slug>](<path>) — skipped — <reason, what replaces it>

## Completed

- [02-feat-profile](v0.1/02-feat-profile.md) — done <YYYY-MM-DD> — <one-line outcome>
````

**Rules**:

- **One line per macro-task**, in every section. The details are in
  the ledger; never copy them here.
- A line moves between sections as the state changes.
- `Completed` grows freely; one line per task keeps it cheap.

### `.docs/tasks/task-log.md` — template

````markdown
# Task log — <service-name>

> Chronological, append-only, one line per event (≤ 200 chars).
> Task IDs are stable; this log is the real timeline.

- <YYYY-MM-DD> — 03-feat-auth — started — <note, e.g. "ahead of 02">
- <YYYY-MM-DD> — 03-feat-auth — wrapped up — <one-line outcome>
````

**Rules**:

- **Append only** — never edit or delete past entries.
- The `planner` owns it: a `started` line when a macro-task moves to
  `in-progress`, a `wrapped up` line at the end of wrap-up.
- If `tasks.md` and the log disagree, `tasks.md` is authoritative for
  *state*, the log for *chronology*.

### `<id>.md` — ledger template

The ledger holds intent and state. Execution details go to the log file.

````markdown
# <id> — <title>

**Status**: pending | in-progress | wrap-up-pending | done | blocked | skipped
**Version**: <x.y or "backlog" or "—">
**Branch**: <branch name, or "—">
**Created**: <YYYY-MM-DD>   **Done**: <YYYY-MM-DD or "—">

## Summary

> Written at wrap-up, ≤ 30 lines. The only section a later reader needs:
> it feeds the pull request body and the CHANGELOG.

- **Delivered**: <what now exists, 2-5 bullets>
- **Divergences from plan**: <bullets, or "none">
- **Decisions promoted**: <ED-xx → architecture.md §y / modules/<n>.md>
- **Quality**: <tests N green (baseline M) · CI <green | not enabled> · review <pass | pass-with-fixes>>
- **Open**: <SI-xxx, follow-up tasks, or "none">

## Plan

> Written in architecture mode (`architect-service`, macro-task planning
> mode). Immutable after approval. 100-300 lines.

### Objective
### Scope
**In scope**: <...>
**Out of scope**: <...>
### Architecture
### Components
### Flows
### Integration
### Design decisions
**D-01 — <title>**: ...
### Risks and open questions
### Approach

## Micro-tasks

> Status moves pending → in-progress → done (or blocked / skipped).
> Descriptive fields are immutable after planning.

### mt-01 — <short title>

- **Status**: pending
- **Security-relevant**: true | false
- **Objective**: <one line>
- **Acceptance criteria**:
  - <bullet>
- **Tests required**: <unit / integration / e2e — be specific>
- **Likely impacted files**: <list, best-effort>
- **Depends on**: <mt-NN or "none">
- **Risk note**: <optional — unverified assumption + plan B>

### mt-02 — <short title>

- **Status**: blocked
- **Reason**: <one line — what blocks, who unblocks>
- …

## Emergent decisions

> Lazy section — empty after planning. Each entry ≤ 10 lines and cites
> the part of the Plan it revises or works around.

### ED-01 — <short title>

- **Status**: active | superseded by <ref>
- **Date / trigger**: <YYYY-MM-DD> — <mt-NN or detailing session>
- **Context**: <1-2 sentences: what was surprising>
- **Decision**: <what was chosen>
- **Alternatives**: <1-3 items considered>
- **Impact**: <what this affects going forward; cross-service: yes/no>

## Wrap-up

> Filled at wrap-up by the planner: outcome lines only. The full
> aggregate review outputs go to the log file.

- **Aggregate code review**: <pass | pass-with-fixes (N fixes) — see log>
- **Aggregate security review**: <pass | findings SI-xxx | not needed>
- **Full test suite**: <N green>
- **CI gate**: <green | not enabled>
- **Architecture consolidation**: <modules/<n>.md created/updated, architecture.md §x>
- **Doc-sync verification**: <ok | fixed: …>
````

**Rules for the ledger**:

- Section order: Summary, Plan, Micro-tasks, Emergent decisions, Wrap-up.
- The Plan is **immutable** after approval. Changes go into Emergent
  decisions, never into the Plan.
- Emergent decisions use the `ED-` prefix (the Plan uses `D-`), so a
  reference like `ED-03` is unambiguous. Numbering is stable: superseded
  entries stay.
- **Never delete** entries. Superseded decisions and blocked micro-tasks
  stay.

### `<id>.log.md` — execution log template

Append-only. Written by the builder and the reviewers; read by the
planner **only at wrap-up** (and, during detailing, only the entries of
the micro-tasks the current one depends on). Its size never weighs on
the per-session context.

````markdown
# <id> — execution log

## mt-01 — <YYYY-MM-DD>

### Completion log (builder)

- **Status**: completed | partial | blocked
- **Files touched**: <paths>
- **Tests**: <added/updated N; ran <subset> — all pass>
- **Build / quality gate**: <pass | fail (…) | not configured>
- **Decisions**: <none | ED-xx>
- **Docs updated**: <files, or "none required">
- **Issues / follow-up**: <only if partial/blocked>

### Security review (security-reviewer)

**Verdict**: pass — <N checks, no findings> | findings: SI-xxx (<severity>), …
<one block per finding, ≤ 5 lines; full detail lives in security-issues.md>

### Code review (code-reviewer)

**Verdict**: pass | pass-with-fixes — <fixes, one line each>

## Wrap-up — <YYYY-MM-DD>

### Aggregate code review (code-reviewer)
### Aggregate security review (security-reviewer)
````

**Rules for the log**:

- One `## mt-NN — <date>` block per micro-task, in execution order. All
  headings of the block are `###` under it — never a top-level `##`
  for a single review.
- **A clean review is one line.** Write details only for findings and
  fixes. The completion log stays within 15 lines.
- Never rewrite past entries — they are history.

### Wrap-up compaction (planner, at every wrap-up)

Run as part of wrap-up, after the reviews and the consolidation:

1. Write the ledger **Summary** (≤ 30 lines) and the **Wrap-up**
   outcome lines.
2. Replace the macro-task line in `tasks.md` with its one-line
   `Completed` form.
3. Append the `wrapped up` line to `task-log.md` (≤ 200 chars).
4. Check that the **living docs** (`specs.md`, `architecture.md`,
   `modules/*.md`) describe the current state only: no "changed in
   task NN" notes, no superseded alternatives. History stays in the
   ledger; the living docs point to it at most once.
5. Leave the log file as it is: after wrap-up nobody loads it again
   unless investigating.

### Version close (versioning only)

When `principal` closes a version (`git-workflow` skill), the CHANGELOG
entry for the service is built from the **Summary → Delivered** bullets
of the version's ledgers. No new prose: the Summaries are the source.

## Decomposition workflow

When invoked to decompose a macro-task into micro-tasks:

1. **Read the macro-task ledger**: `<id>.md` —
   focus on the `Plan` section. Decomposition flows directly from the
   `Approach` subsection of the plan.
2. **Read any existing service docs** (specs, architecture, relevant
   module docs) to understand context.
3. **List candidate micro-tasks** in chat first, before writing anything.
   Discuss with user if helpful.
4. **Apply atomicity criteria** to each candidate. Split if needed.
5. **Assign security-relevant flags** by walking the flag criteria.
6. **Identify dependencies** between micro-tasks. A linear chain is fine;
   DAG is fine; cycles are a sign of bad decomposition.
7. **Write the `Micro-tasks` section** of the ledger
   (`<id>.md`), one subsection per micro-task, all in
   `pending` status. The Plan section stays untouched (it was written
   in architecture mode and is now immutable).
8. **Leave `Emergent decisions` empty**. That section is populated
   lazily, later, when detailing or implementation surfaces an
   emergent decision. If during decomposition you find yourself wanting
   to record a significant design choice, it means the Plan is
   incomplete — propose revising the Plan instead of writing into
   Emergent decisions.

## Detailing workflow (just-in-time)

When the planner is about to hand off a micro-task to the builder:

1. **Read the macro-task ledger** (`<id>.md`) — specifically:
   - The Plan section for architectural intent
   - The Micro-tasks section to find the target micro-task
   - The Emergent decisions section (may contain prior discoveries
     that affect this micro-task)
   - From `<id>.log.md`, **only** the entries of the micro-tasks the
     target one depends on — never the whole log
2. **Elaborate the plan in chat** — concrete implementation steps,
   edge cases, exact files to touch, specific test scenarios.
3. **Do not write this plan to disk**. It lives in the session context
   and is consumed by the builder when the user switches mode.
4. **If detailing surfaces an emergent decision** that changes an
   assumption from the Plan or introduces a workaround, append an
   entry to the `Emergent decisions` section of the ledger. The entry
   must cite which aspect of the Plan is being revised or worked
   around — don't create silent contradictions. If the emergent
   decision is significant enough that the Plan itself should be
   revised, propose this to the user rather than logging a
   contradiction.
5. **Set micro-task status to `in-progress`** in the Micro-tasks
   section of the ledger before handoff.

## Health check (invoked by planner in triage / wrap-up)

- `tasks.md`: check `Completed` section length; rarely needs action.
- Macro-task ledger Micro-tasks section: look for stuck `in-progress`
  (session abandoned?); `blocked` with stale reasons (> 7 days —
  surface to user).
- Macro-task ledger Emergent decisions section: flag entries >10 lines
  — propose promotion to architecture docs.

## When to stop

Decomposition is complete when:

- Every candidate micro-task passes atomicity
- All have acceptance criteria and test scope
- Security-relevant flags are assigned
- Dependencies are clear (no cycles)
- User has reviewed and agreed

If the decomposition produces more than ~10 micro-tasks, the macro-task
is likely too big — consider splitting it into two macro-tasks and
re-decomposing.
