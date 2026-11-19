# Service — <service-name>

> SDD workspace for this service. Project-level context (PDR,
> cross-service architecture, contracts) is auto-loaded together with
> service-local docs. This file tells you how to operate inside this
> service: agents, workflow, file conventions, and boundaries.

## What this service does

<One paragraph: the service's responsibility, what it owns, what it doesn't.>

For the full functional spec see `.docs/specs.md` (already in your context).

## Tech stack

- **Language / runtime**: <e.g. language version + runtime distribution>
- **Framework**: <primary framework(s) used>
- **Persistence**: <database / storage layer, if any>
- **Build / deps**: <build tool and dependency manager>
- **Test**: <test framework(s) and any integration test tooling>

### Quality checks

The commands that make up this service's **quality gate** (lint, format
check, type-check, tests). When `ci.provider` in `../sdd.config.yaml`
is not `none`, the CI pipeline definition is the source of truth and
this list must mirror it; otherwise this list is the only definition.

- <e.g. `<linter> check .`>
- <e.g. `<formatter> --check .`>
- <e.g. `<test runner>`>

*(Leave empty if not defined yet — the gate then runs build + tests only.)*

Skills specific to this service (design system, framework conventions,
testing patterns, etc.) live under `.opencode/skills/`. Agents load
them autonomously based on the skill descriptions when relevant to
the current work.

## Repository layout (this service)

```
.
├── AGENTS.md                              # ← you are here
├── opencode.json                          # instructions, permissions, default_agent
├── .opencode/
│   ├── agents/
│   │   ├── planner.md                     # primary — docs + planning, no source code
│   │   ├── builder.md                     # primary — code + tests
│   │   ├── code-reviewer.md               # subagent — fixes + doc updates
│   │   └── security-reviewer.md           # subagent — read-only + report
│   └── skills/
│       ├── architect-service/             # methodology: internal architecture
│       ├── task-decomposition/            # task IDs, ledger + log, budgets, compaction
│       ├── wrap-up/                       # planner wrap-up procedure
│       ├── git-workflow/                  # branches, PR, CI, merge, tags
│       ├── express-mode/                  # trivial changes
│       ├── free-mode/                     # untracked changes
│       ├── refactor-mode/                 # changes of direction
│       ├── findings-workflow/             # framework findings → skills
│       └── <stack-name>/                  # stack-specific conventions (optional)
├── .docs/
│   ├── specs.md                           # eager (root instructions): every agent
│   ├── security-issues.md                 # mandatory read: security-reviewer
│   ├── architecture/
│   │   ├── architecture.md                # mandatory read: planner, builder, reviewers
│   │   └── modules/
│   │       └── <module-name>.md           # lazy: read when working on module
│   ├── infra/
│   │   ├── server.md                      # mandatory read: planner, builder
│   │   └── ops.md                         # read when touching deploy/ops
│   ├── findings/                          # lazy: raw framework findings log
│   │   └── <framework>.md                 # append-only; see findings-workflow skill
│   └── tasks/                             # lazy: navigate as workflow requires
│       ├── tasks.md                       # macro-tasks index, one line each
│       ├── task-log.md                    # chronological timeline, one line per event
│       ├── v0.2/                          # version folder (only if versioning is on)
│       │   ├── 03-feat-auth.md            # ledger: Summary, Plan, Micro-tasks, ED, Wrap-up
│       │   └── 03-feat-auth.log.md        # execution log (read at wrap-up only)
│       └── backlog/                       # macro-tasks not assigned to a version
└── <source-tree>/                         # service code (layout depends on stack)
```

## Documentation map

**Already in your context** (loaded via root `instructions` in
`opencode.json` — no need to re-read):

| File                                 | Purpose                                         |
| ------------------------------------ | ----------------------------------------------- |
| `.docs/specs.md`                     | What this service does / does not do            |

**Mandatory reads** (lazy by mechanism, mandatory by role — each agent
reads these with its read tools at session/task start, per the
**Context partitioning** table below):

| File / pattern                        | Read when…                                      |
| -------------------------------------- | ----------------------------------------------- |
| `../sdd.config.yaml`                  | At session start (planner, builder): CI, git and versioning settings |
| `../_project/.docs/PDR.md`            | Planning work that touches requirements         |
| `../_project/.docs/architecture.md`   | Cross-service architecture                      |
| `../_project/.docs/contracts/*`       | Service contracts (interfaces to others)        |
| `.docs/architecture/architecture.md`  | Internal architecture                           |
| `.docs/infra/server.md`               | Server, env, configuration                      |
| `.docs/infra/ops.md`                  | Deployment / ops                                |
| `.docs/security-issues.md`            | Open & recently-closed security issues          |

**Lazy-loaded** (read when relevant):

| File / pattern                         | Read when…                                      |
| -------------------------------------- | ----------------------------------------------- |
| `.docs/architecture/modules/*.md`      | Working on the corresponding module             |
| `.docs/tasks/tasks.md`                 | Picking up work / checking macro-task state     |
| `.docs/tasks/task-log.md`              | Chronological timeline of task events (append-only) |
| `.docs/tasks/[v<x.y>/]<id>.md`          | Macro-task ledger: Summary, Plan, Micro-tasks, Emergent decisions, Wrap-up |
| `.docs/tasks/[v<x.y>/]<id>.log.md`      | Execution log — append-only; read at wrap-up (and dependencies at detailing) |
| `.docs/express-log.md`                 | Auditing recent quick fixes made in express mode |

> **Never load whole contracts.** A single `external-<provider>.yaml`
> can be hundreds of KB. Read the index/sections you need, not the
> whole file.

## Context partitioning

`instructions` exists **only at the root** of `opencode.json`. There is
no per-agent variant: an `instructions` key inside an `agent` block is
invalid config — opencode forwards it to the model provider as an
unknown option (silent no-op or hard error, depending on provider).
To change what an agent reads, edit the tables in this file and the
agent's `.opencode/agents/<name>.md`. Keep the root `instructions`
list minimal: it is injected into every agent — internal ones included
(`title`, `compaction`).

Context partitioning is achieved through **mandatory reads**: each
agent reads its docs with normal read tools at session/task start.

| Agent              | Mandatory reads at start                                            |
| ------------------ | -------------------------------------------------------------------- |
| `planner`          | `../sdd.config.yaml`, service arch, server; PDR / contracts / ops / security-issues when the session touches them |
| `builder`          | `../sdd.config.yaml`, service arch, server                           |
| `code-reviewer`    | service arch                                                         |
| `security-reviewer`| service arch, security-issues                                        |

Documents outside an agent's mandatory set can still be read on demand
when the work requires it. The partition optimizes the default state,
not the maximum reachable state. Never point a root `instructions`
glob at `../_project/.docs/contracts/` — external provider contracts
can be enormous (see the warning above).

Resist the temptation to load "everything just in case" — it degrades
instruction-following on the relevant work.

## Agents in this service

- **`planner`** (default at session start). Triages session, plans
  tasks, writes documentation, runs wrap-up at macro-task completion.
  Cannot touch source code.
- **`builder`**. Implements micro-tasks: writes code, tests, iterates
  with build/test cycles. Auto-accepts edits within the micro-task
  perimeter. Invokes `@security-reviewer` automatically when the
  micro-task is flagged `security-relevant: true`; `@code-reviewer` on
  explicit request.
- **`@code-reviewer`** — subagent. Reviews builder output, applies fixes
  directly to code and docs. On demand per micro-task, mandatory at
  wrap-up (aggregate review of the macro-task).
- **`@security-reviewer`** — subagent. Read-only. Produces a security
  report inline in the micro-task log and updates
  `.docs/security-issues.md`. Invoked only when the micro-task is flagged
  `security-relevant: true`, and in aggregate at wrap-up.

Switch primary agents with `Tab`. Subagents are invoked by the builder
(security-reviewer automatically when flagged, code-reviewer on request)
or by the planner at wrap-up; you can also invoke them manually with
`@name` at any time.

> **Note on workspace separation**: this service is its own OpenCode
> workspace. The project-level workspace (`_project/` in the monorepo)
> has its own dedicated agent (`principal`) for cross-service concerns
> (PDR, cross-service architecture, contracts, infrastructure,
> project-wide scaffolding). This service workspace is independent —
> you interact with `planner` / `builder` here without overlap. To
> work on cross-service concerns, change directory to `_project/` and
> launch a fresh OpenCode session there.

## Configuration sync

Some `.md` files in this service contain **configurable placeholders**
that should reflect actual project values:

- `<service-name>` / `<ms-name>`: the name of this service
- `<stack-name>`: the name of the stack-specific skill folder, if you
  use one (e.g. `springboot`, `fastapi`)

These placeholders are resolved by the **planner agent**: at session
start, before triage, the planner scans service docs for unresolved
or drifting placeholders. If found, it asks for confirmation and
applies substitutions across all affected files atomically. Once
values are stable the scan is silent.

Configuration is **never frozen**: if you decide to rename the service
or change stack, just tell the planner ("we're renaming this service
to X" / "we're switching to FastAPI"). It will rerun the sync with
the new values, replacing the previous substitution everywhere.

**What is NOT a configurable placeholder**: any `<...>` appearing
inside template fences (triple backticks) or inside documented output
structures. Examples include `<n>`, `<short-name>`, `<title>`,
`<date>`, `<file>`, `<reason>`, `<reference>`, `<broker>`, `<event>`,
`<protocol>`, `<module-name>`, `<source-tree>`. These are template
structures the agents fill at runtime when writing actual content
(task entries, completion logs, decision records). They must remain
in `<...>` form for the templates to function.

**Models are NOT configured via this mechanism.** Models for each
agent are declared **once** in your global OpenCode config at
`~/.config/opencode/opencode.json`, not in this project's
`opencode.json`. This keeps personal model preferences out of the
repository. They are not touched by the planner's configuration
sync. Recommended split (from field use): `planner` and `builder` run
high-frequency work (triage, detailing, implementation) and can use a
light/fast model; `code-reviewer` and `security-reviewer` carry the
judgment-heavy gates and deserve a heavy/reasoning model.

## Session behavior

At session start, the `planner` agent is active by default. Its default
behavior is to **perform a triage** of the service state and propose the
next action — without requiring any specific keyword from the user. A
greeting, an empty prompt, or a generic "go / let's start" is enough.

If the user's first message contains specific instructions (e.g. "review
the auth module" or "I want to redesign the persistence layer"), the
planner follows those instructions instead of running the default triage.

## Workflow

The default flow on a fresh session, in `planner`:

### 1. Triage (planner)

Planner checks the state of the service:

- Is the foundational documentation (specs, architecture, infra) in place?
  If not → **setup mode**.
- What's the next non-completed macro-task in `tasks.md`?
- Are micro-tasks defined for that macro-task? If not → **planning mode**.
- What's the next non-completed micro-task? → **detailing**.
- Was the previous macro-task just completed? → **wrap-up mode**.

### 2. Setup mode (planner) — if foundational docs are missing

Invoke `architect-service` skill in **service-setup** mode to
produce/update specs and internal architecture. Module-level docs
(`modules/*.md`) are **not** produced at setup — only listed in
`architecture.md`. They're created as macro-tasks complete.

### 2bis. Brownfield mode (planner) — adopting SDD on existing code

When the service already has a codebase but no SDD documentation,
the user can request **brownfield mode** explicitly with phrases
like "modalità adozione", "brownfield", "adopt SDD on this existing
project". The planner runs a 5-step procedure: scope conversation
with the user, code scan to identify modules and integration points,
invocation of `architect-service` in service-setup mode (with
brownfield framing) to produce baseline `specs.md` and
`architecture.md`, listing of identified modules without creating
`modules/<n>.md` files (lazy adoption — they're created as macro-tasks
touch them), and a final summary marking the docs as "baseline that
reflects current state, to be refined".

Brownfield mode is best-effort. The output is less precise than
greenfield documentation and is expected to be refined over time.

### 3. Architecture mode (planner) — for every new macro-task

Before a macro-task can be decomposed into micro-tasks, it needs an
architectural plan. The planner enters architecture mode and:

- Invokes `architect-service` skill in **macro-task-planning** mode
- Works with the user conversationally to define objective, scope,
  module/feature architecture, design decisions (with alternatives
  and rationale), risks, and approach
- Produces the Plan section of `<id>.md` (creating the ledger if needed)
- Waits for explicit user approval

Architecture mode typically **chains directly into planning mode** in
the same session — once the Plan is approved, decomposition happens
without context switching.

### 4. Planning mode (planner) — decompose plan into micro-tasks

With the Plan approved, the planner invokes `task-decomposition`
skill to break the macro-task into atomic micro-tasks in
the Micro-tasks section of `<id>.md`. Each micro-task is tagged with
`security-relevant: true|false` and includes objective, acceptance
criteria, required tests, and likely impacted files.

The decomposition maps directly to the `Approach` section of
the Plan — micro-tasks are the concrete steps that realize the
approved architecture.

### 5. Detailing & handoff (planner)

For the next non-completed micro-task:

- Load the macro-task ledger: read the Plan section (architectural intent) and the Emergent decisions section
  if it exists (emergent decisions from previous micro-tasks)
- Detail the micro-task plan **in chat context** (not on disk)
- If detailing surfaces an emergent decision, append to
  the Emergent decisions section of the ledger (create lazily if empty), always citing
  which aspect of the Plan is being revised or worked around
- Hand off to user: "plan ready, switch to builder with Tab"

### 6. Implementation (builder)

User switches to `builder`. Builder:

- Implements the change, writes tests as part of the same micro-task
- Iterates with build + module-scoped tests until acceptance criteria
  are met. **Tests run during the micro-task are scoped to the
  affected module(s)** (path-based or pattern-based selection), plus
  any tests explicitly listed in the micro-task's "Tests required"
  field. Full suite runs at wrap-up, not per-micro-task.
- Applies any documentation updates triggered by the change
  (see **Documentation sync rules** below)
- Appends a **completion log** (≤ 15 lines) to the execution log
  `<id>.log.md`, documenting the test subset that was run

### 7. Review chain (builder → subagents)

When builder has finished the implementation and completion log:

- If the micro-task is flagged `security-relevant: true`, the builder
  announces: "Micro-task is security-relevant. Invoking
  @security-reviewer. Interrupt to skip." Then invokes
  `@security-reviewer`, which runs read-only analysis, appends a
  **security review** subsection to the current micro-task's block in
  the execution log `<id>.log.md`, and updates `.docs/security-issues.md` if new issues
  are found.
- **Code review is on demand**: per-micro-task code review is **not
  invoked automatically** — with passing tests, forward progress is
  preferred. The user can request it at any time ("run code review")
  or invoke `@code-reviewer` directly; the code-reviewer reviews diff +
  plan adherence, applies fixes directly to code and docs, and appends
  a **review changes** subsection to the current micro-task's block in
  the execution log `<id>.log.md`.
- Aggregate code quality is reviewed **at the macro-task wrap-up**
  (mandatory — see step 10).

### 8. Commit proposal

Once the review chain has passed and the micro-task is marked `done`
(and, if applicable, the macro-task is marked `wrap-up-pending`), the
builder proposes a git commit:

- Gathers the list of files actually touched (from the completion log
  plus any reviewer-applied changes)
- Constructs a commit message in **Conventional Commits** format
  (`type(scope): summary`) with a mandatory footer referencing
  `Task:` and `Micro-task:`
- Shows the proposed message and the files to be staged
- Waits for the user's confirmation

On confirmation, the builder runs `git add -- <files> && git commit -m "<message>" -- <files>`
(explicit paths, never `-A`). The commit hash is reported in the final
summary.

If the user declines or prefers to commit manually, the builder skips
the commit and leaves the working directory ready. The user commits on
their own terms.

The builder never pushes. Integration (push, pull request, CI, merge)
happens once per macro-task, at the end of wrap-up — see **Git
workflow** below.

### 9. End of session

After the commit step, the session ends. **Do not start the next
micro-task automatically** — open a fresh session for the next one.

> **End-of-session commit gate (canonical).** This is the single
> definition of the commit gate; agents reference it rather than restating
> it. **No session that modified files ends without proposing a commit.**
> Procedure: gather the files actually touched; build a Conventional
> Commits message (`type(scope): summary`) with a footer referencing the
> relevant `Task:`/`Micro-task:`/ledger; show the message and the files to
> stage; wait for confirmation; on confirmation run
> `git add -- <files> && git commit -m "<message>" -- <files>` (explicit paths, never `-A` or
> `.` — other services may share the checkout), then report the hash;
> if the user declines, skip and leave the tree ready. Never push during
> a session: push, pull request and merge happen only in the
> integration step at the end of wrap-up (`git-workflow` skill). This gate applies to every agent (`planner`, `builder`,
> `principal`) and is most often skipped at two points: **after a
> macro-task Plan is approved** (planning mode) and at the **end of
> wrap-up** — do not skip it there.

### 10. Wrap-up mode (planner) — at macro-task completion

When triage detects a macro-task with status `wrap-up-pending`, planner:

- Reads the macro-task ledger (Plan, Micro-tasks, Emergent decisions)
  and its execution log `<id>.log.md` to
  build an aggregate view of what was built vs what was planned
- Invokes `@code-reviewer` in **aggregate wrap-up mode** to review
  the cumulative diff of the entire macro-task — catches cross-cutting
  issues invisible to per-micro-task review
- **Runs the full test suite** to verify no cross-functional
  regressions. If any test fails, the wrap-up is **blocked**: the
  macro-task stays at `wrap-up-pending` until a follow-up micro-task
  fixes the regression
- **Invokes `architect-service` skill in consolidation mode** to
  update or create `modules/<n>.md` for modules affected by the
  macro-task. Stable decisions from the Plan section are promoted to
  module docs; the Plan itself remains as historical record.
- Reviews the Emergent decisions section if non-empty (emergent decisions
  logged during detailing/implementation). Decides for each entry:
  leave as historical record, promote to architecture if it has
  lasting implications, or flag as a lesson for future planning if
  it represents a plan-revision gap.
- Cross-checks `.docs/security-issues.md` for issues opened by this
  macro-task
- **Doc-sync cross-check**: extracts all "Documentation updates" entries
  from the micro-task logs and verifies the corresponding files reflect
  those changes; proposes fixes if discrepancies are found
- Scans for residual dead code across the aggregate diff
- Transitions macro-task status `wrap-up-pending → done`
- **Integrates the branch** per `sdd.config.yaml` (`git-workflow`
  skill): push, pull request, CI checks, merge on user approval — or
  a local merge, or nothing, depending on `git.remote`
- Surfaces the next macro-task

## Git workflow

Driven by the `git` and `ci` sections of `../sdd.config.yaml` and
executed through the `git-workflow` skill:

- **Macro-task start** (planning mode): the planner creates the task
  branch (`branch_per: task`) or joins the version branch
  (`branch_per: version`).
- **During micro-tasks**: commits only, with explicit paths. No branch
  switch, merge, rebase or stash — other services may share the
  checkout.
- **End of wrap-up**: integration — push, pull request, CI checks,
  merge on user approval (GitHub), or the equivalent for `other` /
  `none`.
- **Version close** (`versioning.enabled: true`): owned by `principal`
  — changelog, release merge, tag.

## UI work — mockups are the source of truth (frontend services only)

> **Applies only if this service has a UI.** If the service is headless
> (backend, worker, API-only), ignore this section entirely.

For a frontend service, UI mockups are the highest-fidelity source of truth
for anything visual. Before writing or planning code for a screen/view, the
`planner` (detailing) and `builder` (implementation) **must**:

1. Read `../_project/.docs/mockups/index.md` to locate the prototype for the
   screen (the `principal` maintains this index).
2. **If the screen isn't listed, do NOT conclude "no mockup"** — the UI may
   be embedded in a related prototype (a form inside a wizard, a sheet inside
   a list). Inspect adjacent flow prototypes before deciding one doesn't
   exist.
3. Load the **HTML prototype** — it is the richest source: exact copy,
   spacing, sizes, border radius, colors, hierarchy, and interactions. Any
   PNG is complementary and optional (the model may not read images).
4. **Copy texts and labels verbatim.** Do not paraphrase, translate, or
   "improve" them. Extract concrete values (text, font, spacing, sizes,
   colors via the theme — not hard-coded) and apply them. The mockup always
   wins.

Skipping the mockup and inventing UI is a defect: it produces wrong copy,
wrong spacing, and rework. Reading the prototype first is the cheapest path
to a correct screen.

## Session discipline

**One session = one micro-task** (default). After implementation + review,
end the session. The user opens a new one for the next micro-task.

Exceptions are allowed when explicitly requested by the user (e.g. "do
micro-task 3 and then 4 if it's trivial"). In that case ask confirmation,
do both, then stop. Never chain silently.

> The workflow above is the **default**. User instructions during the
> conversation always take precedence — if the user wants to skip a step,
> revisit a doc, or restructure a task, follow their lead without
> lecturing them about the flow.

## Express mode

For trivial work — typo fixes, dependency bumps, config tweaks, minor
renames — planner and builder support **express mode**: no micro-task
decomposition, no documentation tracking, but build + tests and the
quality gate stay mandatory and a commit is proposed. Express is **not**
allowed for anything touching specs, architecture, contracts, security,
persistence schema or public interfaces. The only record is one line in
`.docs/express-log.md`. Full procedure: `express-mode` skill.

## Free mode

Free mode is the user's explicit choice to skip SDD entirely: switch
straight to `builder` with `Tab` and give an implementation request. The
builder asks for confirmation, implements, optionally runs tests and
proposes a commit — with **no** plan, ledger, review, doc sync or log
line. Use it for experiments and personal tweaks, never for work someone
will need to audit. Full procedure: `free-mode` skill.

## Documentation sync rules

Documentation is **source of truth** and must stay aligned with the code
at all times. Every agent that modifies code is responsible for propagating
the change to the relevant documentation in the same micro-task.

**Living docs describe the current state only.** `specs.md`,
`architecture.md`, `modules/*.md` and `infra/*` say what the service *is*,
not how it got there: rewrite the affected passage instead of appending
"changed in task NN" notes or keeping superseded alternatives. History
lives in the ledgers; a living doc may point to one ledger, once, when the
reason behind a choice matters.

### Triggers — when to update what

| If the change introduces/modifies…                  | Update…                                           |
| --------------------------------------------------- | ------------------------------------------------- |
| Runtime or framework version                        | `AGENTS.md` (Tech stack), `.docs/infra/server.md` |
| A new architecturally-significant dependency        | `AGENTS.md` (Tech stack)                          |
| Persistence technology / schema                     | `.docs/architecture/architecture.md` + relevant module doc |
| A new module or major flow                          | `.docs/architecture/modules/<n>.md` (new or updated) |
| Behavior described in specs                         | `.docs/specs.md`                                  |
| Configuration parameters, environment variables     | `.docs/infra/server.md`                           |
| CI/CD pipeline changes                              | `.docs/infra/ops.md`                              |
| Public service interface (request/response shape)   | `../_project/.docs/contracts/*` — **flag only, do not modify** (see Boundaries) |

### How updates are enforced

- **Builder** declares documentation updates in the completion log under
  a "Documentation updates" field — must always be present, even if
  "none required". This makes the check explicit.
- **Code-reviewer** verifies the declared updates are coherent with the
  code diff and fixes discrepancies directly.
- **Planner** in wrap-up mode performs a global cross-check: reads the
  "Documentation updates" entries from all micro-task logs of the closing
  macro-task and verifies the actual files match.

### Cross-service changes

Any change that impacts another service (contract changes, event
schemas, shared interfaces) must **not** be modified from inside this
service. Instead:

- Flag the required change in the ledger's Emergent decisions section with a clear
  description
- Stop and surface the issue to the user
- The change is then handled from the `_project/` workspace in a separate session
  (`principal` updates `../_project/.docs/contracts/*` and
  coordinates impacted services)

## Handling significant changes

Specs shifting, patterns replaced, past decisions invalidated: these are
part of the lifecycle. A **narrow** change is normal re-planning. A
**medium** one (modules of this service, invalidates past decisions)
becomes a macro-task of type `ref`, created after an impact mapping that
marks superseded decisions and abandoned work — history is never
deleted. A **wide** one (service-level or cross-service) starts from the
`_project/` workspace. Full procedure: `refactor-mode` skill.

## Dead code & legacy hygiene

LLM-driven implementation has a known failure mode: leaving behind dead
code, legacy stubs, parallel implementations, deprecated functions, or
commented-out blocks "just in case". Over time this accumulates and
makes the codebase harder to reason about for both humans and the agents
themselves.

This service enforces an active cleanup discipline.

### The default rule

When the builder modifies code in a way that **replaces** existing
behavior — refactoring, renaming, migrating, swapping an implementation —
the old code is **removed in the same micro-task**. No parallel
implementations, no `oldX`/`newX` coexistence, no orphaned helpers.

### When kept code is intentional — the `SDD-KEEP` marker

Sometimes code that looks dead is intentionally retained: backward
compatibility, feature-flagged paths, parallel implementations during
multi-step migrations, public API surfaces called from outside the
service. These cases are legitimate but **must be marked**.

Use a standardized comment in the code, in the syntax appropriate for
the language:

```
SDD-KEEP: <reason>, see <reference>
```

Examples:

- `SDD-KEEP: backward compat for v1 API, remove after v2 migration (see 12-ref-services ledger, ED-07)`
- `SDD-KEEP: feature-flagged, activated by FEATURE_X (see specs.md)`
- `SDD-KEEP: temporary parallel impl during migration to NewService, remove in 12-ref-services`

The marker must include a **reason** and a **reference** to a doc or
task that justifies the retention. A naked `SDD-KEEP` with no reason is
treated as missing.

### Detection and removal policy

Both the builder and the code-reviewer apply this policy.

**Cleared for automatic removal** (no question asked):
- Imports that are no longer referenced
- Local variables that are written but never read
- Private/file-local functions never called within the file
- Commented-out code blocks (any age)
- Dead branches after a refactor (`if (false) { ... }`, unreachable
  `else`, etc.)

**Flagged for decision** (do not remove silently):
- Public functions / exported types that appear unused within the
  service — they may be part of the service's external surface
- Code paths reachable only via configuration or feature flags
- Any code carrying an `SDD-KEEP` marker — verify the reason is still
  valid and act accordingly

When the code-reviewer finds flagged-for-decision dead code without an
`SDD-KEEP` marker, it surfaces the question with a `Verdict: needs-decision`
in its review section. The user decides.

### Where this is enforced

- **Builder**: cleans up as part of the implementation; declares any
  `SDD-KEEP` markers added in the completion log
- **Code-reviewer**: actively detects dead code in the diff; auto-removes
  the safe categories, flags the ambiguous ones
- **Planner (wrap-up mode)**: surveys the macro-task aggregate diff for
  dead code patterns left across multiple micro-tasks

## Conventions & findings — where lessons go

When a lesson emerges (a recurring mistake, a discovered gotcha, a rule worth
keeping), route it by its nature — **do not dump everything into this file**.
Three destinations:

1. **Universal SDD process lesson** (true for any project — e.g. "read the
   mockup before UI code", "no wrap-up with red tests") → it belongs in the
   **pattern itself** (the agent prompts / skills), not in a per-project
   registry. Surface it so it can be upstreamed; don't record it here.
2. **Project-specific *flow* convention** (a process rule peculiar to *this*
   project, not tied to a framework) → **Project flow rules** below. Keep it
   short.
3. **Framework/library finding** (a `<stack-name>` gotcha, version bug,
   workaround) → the **findings workflow**, never here. See the
   `findings-workflow` skill: findings live in `.docs/findings/<framework>.md`
   and, once recurring, in a `<framework>-findings` skill loaded on demand.

The test: *is it framework-specific?* → findings skill. *Is it universal
SDD?* → the pattern. *Only what's left* — project-level process discipline —
lands in Project flow rules. This keeps `AGENTS.md` lean by construction.

### Project flow rules

Project-specific **process** conventions that are neither universal SDD rules
nor framework-specific. Keep this short — if it grows, something in it is
probably category 1 (upstream to the pattern) or category 3 (move to a
findings skill).

- FR-01 — <short title>: <the rule> (emerged in <session>)

*(empty at bootstrap — expected to stay short)*

## File conventions & anti-bloat strategy

This service maintains several incremental files. Each has a defined
growth strategy to prevent bloat. Skills and agent prompts contain the
strict templates; this section is the high-level reference.

| File                                | Owner             | Growth strategy                                      |
| ----------------------------------- | ----------------- | ---------------------------------------------------- |
| `tasks.md`                          | planner           | **One line per macro-task** in every section; details live in the ledger |
| `task-log.md`                       | planner           | Append-only timeline, one line (≤ 200 chars) per event; never reorder |
| `<id>.md` (macro-task ledger)       | planner (+ builder for ED entries) | Summary (≤ 30 lines, at wrap-up), Plan (immutable, 100-300 lines), Micro-tasks, Emergent decisions (≤ 10 lines each), Wrap-up (outcome lines). Read at detailing and wrap-up |
| `<id>.log.md` (execution log)       | builder + reviewers | Append-only. Completion log ≤ 15 lines; a clean review is one line. Read at wrap-up, and at detailing only for the micro-tasks the current one depends on |
| `.docs/security-issues.md`          | security-reviewer | Open + recently-closed only; archive older closed   |
| `.docs/findings/<framework>.md`     | builder           | Append-only raw framework findings; promote to `<framework>-findings` skill at 2 hits. See `findings-workflow` skill |
| `.docs/architecture/architecture.md`| planner           | Split into `modules/<x>.md` if > ~300 lines         |
| `.docs/architecture/modules/<n>.md` | planner           | Created at wrap-up of macro-task introducing the module; updated at subsequent wrap-ups of macro-tasks touching it |
| `.docs/express-log.md`              | builder           | Append-only, one line per fix. Archive older entries manually if needed |

Planner runs lightweight **health checks** during triage and surfaces
suggestions (consolidate, split, archive) — never executes them silently.

## Boundaries

- **Do not** modify project-level docs (`../_project/.docs/*`) directly from this
  level. Cross-service changes must be flagged in the ledger's Emergent decisions section
  — see **Documentation sync rules → Cross-service changes** above.
- **Do not** skip the planner → builder → review chain without explicit
  user instruction.
- **Do** keep incremental files structured and short. Narrative belongs
  in chat, not in committed files.
- **Do** check that a file exists with a read/stat on its **exact path**
  before writing it — never with a glob (globs may skip dot-folders such
  as `.docs/`). An existing file is extended, never recreated.
