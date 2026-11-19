---
name: task-decomposition
description: Methodology for breaking macro-tasks into atomic micro-tasks. Use this skill when a macro-task is ready to be broken down (the macro-task ledger task-NN-<n>.md exists with an approved Plan section but no Micro-tasks section yet), when revising an existing decomposition, or when creating/updating the service's tasks.md index. Enforces atomicity, testability, and the security-relevant flag for downstream review routing. Operates on the consolidated macro-task ledger (single file per macro-task, sections: Plan, Micro-tasks, Emergent decisions, Execution logs, Wrap-up).
---

# Task-Decomposition — From Macro to Micro

You are in **task-decomposition mode**. Your job is to decompose a macro-
task into atomic micro-tasks and to maintain the task documentation
files. The pattern uses **two artifacts**:

- `.docs/tasks/tasks.md` — index of macro-tasks across the service
- `.docs/tasks/task-NN-<n>.md` — single ledger per macro-task,
  containing Plan, Micro-tasks, Emergent decisions, Execution logs,
  and Wrap-up sections in one file

Decomposition operates on the macro-task ledger: it reads the Plan
section (already written) and produces the Micro-tasks section.

## Operating principles

1. **Start from the plan**. Every macro-task has an approved Plan
   section in its ledger (`task-NN-<n>.md`), written by the
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

## Naming convention for refactor macro-tasks

When a macro-task exists to carry out a refactor (typically created by
the planner in refactor mode), use the prefix:

```
task-NN-refactor-<short-name>
```

Example: `task-07-refactor-persistence-layer`.

The `refactor` prefix makes these tasks recognizable at a glance in
`tasks.md`, which matters because they have special semantics:

- They invalidate or supersede earlier decisions — a reader should
  know immediately that this task was corrective, not greenfield
- They often introduce `SDD-KEEP` markers for staged migrations that
  reference the closure of this specific task
- Their Plan section is the source of truth for the target architecture
  until the wrap-up updates `specs.md`/`architecture.md`/`modules/*.md`
  to reflect the refactored code

Non-refactor macro-tasks should not use the `refactor-` prefix even
if they happen to touch existing code — reserve the prefix for work
that was created specifically to change direction.

## Output artifacts

The pattern uses **three artifacts** for tracking macro and micro tasks:

1. **`.docs/tasks/tasks.md`** — index of macro-tasks at service level,
   grouped by *state* (Active/Blocked/Skipped/Completed)
2. **`.docs/tasks/task-NN-<name>.md`** — single ledger file per
   macro-task, containing plan, micro-tasks, decisions, and execution
   logs all in one place
3. **`.docs/tasks/task-log.md`** — append-only *chronological* timeline of
   task events (started / wrapped-up), decoupled from the numbering

Why the log exists: task numbers (`task-NN`) are **stable identifiers**, not
an execution order. When you insert an imminent `task-20` while `15–19`
remain planned for later, the numeric order no longer reflects reality. The
log records the real timeline so history stays readable without ever
renumbering.

This is intentionally consolidated. With modern context windows and
file-reading tools, a single rich ledger per macro-task is more
efficient than splitting into multiple small files: fewer tool calls,
no synchronization risk between siblings, and the agent sees the
whole macro-task state in one read.

### `.docs/tasks/tasks.md` — template

````markdown
# Tasks — <service-name>

> Macro-task index. The full ledger of each macro-task lives in
> `task-NN-<name>.md` in the same folder.

## Active

### task-01-<short-name>

- **Status**: in-progress
- **Objective**: <one line — what this macro-task achieves>
- **Depends on**: <other macro-task IDs or "none">
- **Ledger**: `./task-01-<short-name>.md`

### task-02-<short-name>

- **Status**: pending
- **Objective**: …
- **Depends on**: task-01
- **Ledger**: `./task-02-<short-name>.md`

## Blocked

### task-NN-<short-name>

- **Status**: blocked
- **Reason**: <what is blocking, owner of unblock>
- **Objective**: …
- **Ledger**: `./task-NN-<short-name>.md`

## Skipped

### task-NN-<short-name>

- **Status**: skipped
- **Reason**: <why, and what replaces it if anything>

## Completed

### task-NN-<short-name> — <date>

- **Objective**: <original objective>
- **Ledger**: `./task-NN-<short-name>.md`
````

**Rules**:

- Sections (`Active`, `Blocked`, `Skipped`, `Completed`) group by state.
  A task's entry moves between sections as its state changes.
- Numbering is stable — once assigned, a task's NN never changes.
  Skipped tasks keep their number.
- `Completed` section holds history; grow freely. If it becomes
  unreadably long (rare, probably after a year+), archive to
  `tasks-archive.md`.

### `.docs/tasks/task-log.md` — template

````markdown
# Task log — <service-name>

> Chronological, append-only. One entry per task event, newest at the
> bottom. Task numbers are stable IDs; this log is the real timeline.
> Never reorder or renumber — only append.

## <YYYY-MM-DD>

- task-NN-<short-name>: started <note, e.g. "inserted ahead of 15–19, which
  remain planned">
- task-NN-<short-name>: wrapped up — <one-line outcome>
````

**Rules**:

- **Append only** — never edit or delete past entries; the log is history.
- The `planner` owns it: append a `started` entry when a macro-task moves to
  `in-progress` (planning mode), and a `wrapped up` entry at the end of
  wrap-up. Keep each line to one sentence.
- The log complements `tasks.md` (which groups by state); it does not
  replace it. If they ever disagree, `tasks.md` is authoritative for
  *state*, the log for *chronology*.

### `.docs/tasks/task-NN-<name>.md` — ledger template

This is **the** file for a macro-task. Plan, micro-tasks, emergent
decisions, and execution logs all live here.

````markdown
# task-NN-<name>

**Status**: pending | in-progress | wrap-up-pending | done | blocked | skipped
**Created**: <YYYY-MM-DD>
**Last updated**: <YYYY-MM-DD>

---

## Plan

> Architectural plan written by the planner in architecture mode (via
> the `architect-service` skill, macro-task-planning mode). Immutable
> after approval. The full plan template lives in `architect-service`
> skill — this section embeds the result.

### Objective
<...>

### Scope
**In scope**: <...>
**Out of scope**: <...>

### Architecture
<...>

### Components
<...>

### Flows
<...>

### Integration
<...>

### Design decisions
**D-01 — <title>**: ...

### Risks and open questions
<...>

### Approach
<...>

---

## Micro-tasks

> Atomic units of implementation work. Each has its own status that
> moves pending → in-progress → done (or blocked / skipped).

### micro-task-01-<short-name>

- **Status**: pending
- **Security-relevant**: true | false
- **Objective**: <one line>
- **Acceptance criteria**:
  - <bullet 1>
  - <bullet 2>
- **Tests required**: <unit / integration / e2e — be specific>
- **Likely impacted files**: <list, best-effort>
- **Depends on**: <other micro-task IDs or "none">
- **Risk note**: <optional — unverified assumption + plan B>

### micro-task-02-<short-name>

- **Status**: pending
- **Security-relevant**: false
- **Objective**: …
- **Acceptance criteria**: …
- **Tests required**: …
- **Likely impacted files**: …
- **Depends on**: micro-task-01

### micro-task-NN-<short-name>

- **Status**: blocked
- **Reason**: <one line — what blocks, who unblocks>
- **Objective**: …

---

## Emergent decisions

> Lazy section — leave empty after planning. Populated only if
> discoveries surface during detailing or implementation. Each entry
> must cite which aspect of `Plan` it revises or works around.

### ED-01 — <Short title>

- **Status**: active | superseded by <ref>
- **Date**: <YYYY-MM-DD>
- **Trigger**: <which micro-task or detailing session surfaced this>
- **Context**: <1-2 sentences: what was surprising>
- **Decision**: <what was chosen in response>
- **Alternatives**: <bullet, 1-3 items considered>
- **Impact**: <who/what this affects going forward>
- **Cross-service**: <true/false>
- **Superseded because**: <only if Status is superseded>

---

## Execution logs

> Append-only. One subsection per micro-task as it's executed. Builder
> writes the completion log; reviewers append their findings.

### micro-task-01-<short-name> — <YYYY-MM-DD>

#### Completion log (builder)

- **Status**: completed | partial | blocked
- **Files touched**: <list>
- **Tests**: <count, pass/fail>
- **Build run**: <yes/no, outcome>
- **Decisions**: <inline notes; promote to "Emergent decisions" above if significant>
- **Documentation updates**: <files updated, or "none required">
- **Issues / follow-up**: <if Status is partial or blocked>

#### Security review (security-reviewer)

(Only present if micro-task was `security-relevant: true`. The
security-reviewer is read-only and appends findings here.)

#### Code review (code-reviewer)

(Only present if user explicitly invoked code review for this
micro-task. The code-reviewer may apply fixes to source/test files
and appends the review summary here.)

### micro-task-02-<short-name> — <YYYY-MM-DD>

(Repeat structure for each executed micro-task.)

---

## Wrap-up

> Filled at wrap-up time by the planner. Summarizes the macro-task
> closure activities.

### Aggregate code review (code-reviewer)

(Mandatory at wrap-up. Cumulative review of the entire macro-task
diff, catching cross-cutting issues.)

### Architecture consolidation

- **modules/<n>.md created/updated**: <list>
- **architecture.md updates**: <list>
- **Decisions promoted**: <list of plan decisions promoted to module docs>

### Doc-sync verification

(Result of cross-checking that documentation updates declared in
completion logs were actually applied.)

### Security issues opened

(List of security-issues.md entries opened during this macro-task that
remain `open` at closure time.)

### Final notes

(Brief summary of the macro-task as completed: any divergences from
plan, lessons learned, follow-up macro-tasks suggested.)
````

**Rules for the ledger**:

- Sections live in this order: Plan, Micro-tasks, Emergent decisions,
  Execution logs, Wrap-up. Don't shuffle.
- The Plan section is **immutable** after approval. Changes go into
  Emergent decisions, never into Plan.
- The Execution logs section is **append-only**. Never rewrite past
  entries — they're history.
- Micro-tasks section can have status updates (pending → in-progress
  → done) but the descriptive content (Objective, Acceptance, Tests)
  is also immutable after planning.
- Numbering is stable, and **emergent decisions use the `ED-` prefix**
  (distinct from the Plan's `D-` design decisions, so a reference like
  `ED-03` is unambiguous). Once `ED-01` exists, a new emergent decision
  is `ED-02`, even if `ED-01` becomes superseded.
- **Never delete** entries. Superseded decisions stay; blocked
  micro-tasks stay; old execution logs stay.

## Decomposition workflow

When invoked to decompose a macro-task into micro-tasks:

1. **Read the macro-task ledger**: `.docs/tasks/task-NN-<n>.md` —
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
   (`task-NN-<n>.md`), one subsection per micro-task, all in
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

1. **Read the macro-task ledger** (`task-NN-<n>.md`) — specifically:
   - The Plan section for architectural intent
   - The Micro-tasks section to find the target micro-task
   - The Emergent decisions section (may contain prior discoveries
     that affect this micro-task)
   - Recent Execution logs entries (what other micro-tasks of this
     macro-task have already done)
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
