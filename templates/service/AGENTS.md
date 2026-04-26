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

Skills specific to this service (design system, framework conventions,
testing patterns, etc.) live under `.opencode/skills/`. Agents load
them autonomously based on the skill descriptions when relevant to
the current work.

## Repository layout (this service)

```
.
├── AGENTS.md                              # ← you are here
├── opencode.json                          # eager-loaded context + default_agent
├── .opencode/
│   ├── agents/
│   │   ├── planner.md                     # primary — heavy model, doc + planning
│   │   ├── builder.md                     # primary — light model, code + tests
│   │   ├── code-reviewer.md               # subagent — heavy model, fixes + doc updates
│   │   └── security-reviewer.md           # subagent — heavy model, read-only + report
│   └── skills/
│       ├── architect-service/             # methodology: internal architecture
│       ├── task-decomposition/            # methodology: macro → micro tasks
│       └── <stack-name>/                  # stack-specific conventions (optional)
├── .docs/
│   ├── specs.md                           # eager: planner, builder, reviewers
│   ├── security-issues.md                 # eager: planner, security-reviewer only
│   ├── architecture/
│   │   ├── architecture.md                # eager: planner, builder, reviewers
│   │   └── modules/
│   │       └── <module-name>.md           # lazy: read when working on module
│   ├── infra/
│   │   ├── server.md                      # eager: planner, builder
│   │   └── ops.md                         # eager: planner only (lazy for builder)
│   └── tasks/                             # lazy: navigate as workflow requires
│       ├── tasks.md                       # macro-tasks index with status
│       ├── task-01-name.md                # ledger: Plan, Micro-tasks, Decisions, Logs
│       ├── task-02-name.md                # one ledger file per macro-task
│       └── ...
└── <source-tree>/                         # service code (layout depends on stack)
```

## Documentation map

**Already in your context** (no need to re-read):

| File                                 | Purpose                                         |
| ------------------------------------ | ----------------------------------------------- |
| `../.docs/PDR.md`                    | Project requirements                            |
| `../.docs/architecture.md`           | Cross-service architecture                      |
| `../.docs/contracts/*`               | Service contracts (interfaces to others)        |
| `.docs/specs.md`                     | What this service does / does not do            |
| `.docs/architecture/architecture.md` | Internal architecture                           |
| `.docs/infra/server.md`              | Server, env, configuration                      |
| `.docs/infra/ops.md`                 | CI/CD, deployment                               |
| `.docs/security-issues.md`           | Open & recently-closed security issues          |

**Lazy-loaded** (read when relevant):

| File / pattern                         | Read when…                                      |
| -------------------------------------- | ----------------------------------------------- |
| `.docs/architecture/modules/*.md`      | Working on the corresponding module             |
| `.docs/tasks/tasks.md`                 | Picking up work / checking macro-task state     |
| `.docs/tasks/task-NN-*.md`             | Macro-task ledger: Plan, Micro-tasks, Decisions, Execution logs all in one |
| `.docs/express-log.md`                 | Auditing recent quick fixes made in express mode |

## Context partitioning

Each agent has a **partitioned set of eager-loaded documents**, declared
per-agent in `opencode.json`. This avoids saturating the context of
focused agents (builder, reviewers) with documents that are only
relevant to broad orchestration (planner).

Eager-loaded by agent:

| Agent              | Eager-loaded docs                                                                 |
| ------------------ | --------------------------------------------------------------------------------- |
| `planner`          | PDR, project arch, contracts, specs, service arch, server, ops, security-issues   |
| `builder`          | specs, service arch, server                                                       |
| `code-reviewer`    | specs, service arch                                                               |
| `security-reviewer`| specs, service arch, security-issues                                              |

Documents not eager-loaded for an agent can still be read on demand
with `cat` or other read tools when the work requires it. The partition
optimizes the default state, not the maximum reachable state.

If you customize the agents or add new ones, set their `instructions`
field appropriately in `opencode.json`. Resist the temptation to load
"everything just in case" — it degrades instruction-following on the
relevant work.

## Agents in this service

- **`planner`** (default at session start) — heavy/reasoning model.
  Triages session, plans tasks, writes documentation, runs wrap-up at
  macro-task completion. Cannot touch source code.
- **`builder`** — light/fast model. Implements micro-tasks: writes code,
  tests, iterates with build/test cycles. Auto-accepts edits within the
  micro-task perimeter. Automatically invokes reviewers at completion.
- **`@code-reviewer`** — subagent, heavy/reasoning model. Reviews builder
  output, applies fixes directly to code and docs.
- **`@security-reviewer`** — subagent, heavy/reasoning model. Read-only.
  Produces security report inline in the micro-task log and updates
  `.docs/security-issues.md`. Invoked only when the micro-task is flagged
  `security-relevant: true`.

Switch primary agents with `Tab`. Subagents are invoked automatically by
the builder; you can also invoke them manually with `@name` at any time.

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

**Models and permissions are NOT configured via this mechanism.**
Models for each agent live in `opencode.json` under `agent.<name>.model`
as `<your-heavy-model>` / `<your-light-model>` placeholders. You
substitute them **manually** in `opencode.json`, once, after choosing
your providers. They are not touched by the planner's configuration
sync.

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
- Produces the Plan section of `.docs/tasks/task-NN-<n>.md` (creating the ledger if needed)
- Waits for explicit user approval

Architecture mode typically **chains directly into planning mode** in
the same session — once the Plan is approved, decomposition happens
without context switching.

### 4. Planning mode (planner) — decompose plan into micro-tasks

With the Plan approved, the planner invokes `task-decomposition`
skill to break the macro-task into atomic micro-tasks in
the Micro-tasks section of `task-NN-<n>.md`. Each micro-task is tagged with
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
- Appends a **completion log** subsection to the Execution logs section
  of `task-NN-<n>.md`, documenting the test subset that was run

### 7. Automatic review chain (builder → subagents)

When builder has finished the implementation and completion log, it
**automatically invokes the review chain**:

- Announces in chat: "Implementation complete. Invoking @code-reviewer
  automatically. Interrupt now if you want to skip."
- Invokes `@code-reviewer` via the task tool. Code-reviewer reviews
  diff + plan adherence, applies fixes directly to code and docs, and
  appends a **review changes** subsection within the current micro-task's Execution log.
- If the micro-task is flagged `security-relevant: true`, announces:
  "Code review complete. Micro-task is security-relevant. Invoking
  @security-reviewer. Interrupt to skip." Then invokes
  `@security-reviewer` which runs read-only analysis, appends a
  **security review** subsection within the current micro-task's Execution log, and updates
  `.docs/security-issues.md` if new issues are found.

The user can interrupt between announcement and invocation, or invoke
reviewers manually with `@name` at any time.

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

On confirmation, the builder runs `git add <files>` (selective, never
`-A`) and `git commit -m "<message>"`. The commit hash is reported
in the final summary.

If the user declines or prefers to commit manually, the builder skips
the commit and leaves the working directory ready. The user commits on
their own terms.

The builder never runs `git push` — pushing is always an explicit user
action afterward.

### 9. End of session

After the commit step, the session ends. **Do not start the next
micro-task automatically** — open a fresh session for the next one.

### 10. Wrap-up mode (planner) — at macro-task completion

When triage detects a macro-task with status `wrap-up-pending`, planner:

- Reads the macro-task ledger (Plan + Execution logs) to
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
- Surfaces the next macro-task

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

Not every change deserves the full pipeline. For trivial work — typo
fixes, dependency bumps, config tweaks, minor renames — both planner
and builder support **express mode**, which bypasses micro-task
decomposition and documentation tracking while keeping the code-reviewer
as a safety net.

### When it applies

Express mode is appropriate when **all** of the following hold:

- The change is small and self-contained
- Does not affect specs, architecture, or module docs
- Is not security-relevant (auth, crypto, input validation, secrets)
- Does not touch cross-service contracts
- Does not introduce or remove a module or major flow
- Does not modify persistence schema

If any of these fails, use the normal flow.

### How it's triggered

Express mode is **user-initiated or auto-suggested**. Ways to activate:

- User explicitly asks: "quick fix", "express mode", "modalità express",
  "just a typo"
- User describes a change that the planner recognizes as trivial, in
  which case the planner proposes express mode and waits for confirmation

Never force express mode on a request — always confirm with the user
before proceeding.

### What happens in express mode

- Planner produces a one-line plan in chat (no files created)
- Builder implements the change
- Builder runs build + tests
- Builder invokes `@code-reviewer` (safety net, even for trivial work)
- If code-reviewer modifies code, builder re-runs build + tests
- Builder appends a one-line entry to `.docs/express-log.md`
- Builder proposes a commit (Conventional Commits format, no SDD
  footer, just `Ref: .docs/express-log.md`) and waits for user
  confirmation — same as the normal flow
- Session ends

### What express mode does NOT produce

- No entry in any Micro-tasks section
- No subsection in any Execution logs section
- No `decisions.md` entry
- No status change in `tasks.md`
- No security-review (by constraint, express excludes security-relevant
  changes)

### The express log

`.docs/express-log.md` is a lightweight cronological register. One line
per fix, grouped by date. Example:

```markdown
# Express fixes log

Quick fixes and minor changes that didn't go through the full SDD
pipeline. For substantial changes, see `tasks/`.

## 2026-04-23
- Bumped dep X from A to B (patch) — <manifest file>
- Fixed typo in error message "invlaid" → "invalid" — <file>

## 2026-04-22
- Increased request timeout from 30s to 60s — <config file>
```

The log is lazy-loaded (not in `opencode.json` instructions). Read it
when you need to audit recent quick fixes.

## Free mode

For changes the user wants to make **without any SDD ceremony at
all** — exploratory edits, prototypes, personal tweaks — the builder
also supports **free mode**.

**How to activate**: open a session, switch directly to the `builder`
agent with `Tab` (skipping the planner), and give your implementation
request. The builder detects the absence of a planner-produced plan
and the absence of an "express mode confirmed" announcement, and
treats the request as free mode.

**What free mode does**:
- The builder asks for explicit confirmation before proceeding
- It implements the change as requested
- Optionally runs tests if relevant
- Optionally proposes a commit (Conventional Commits format, no SDD
  task footer)

**What free mode does NOT do**:
- No plan, no micro-task ledger, no `tasks.md` entry
- No documentation sync
- No code-reviewer or security-reviewer invocation
- **No tracking** anywhere — not even an `express-log.md` line. Free
  mode is intentionally untracked: there is no audit trail.
- No status transitions

**When to use it**: when you know what you want to do, the change is
under your direct supervision, and the SDD overhead (even express)
would slow you down. Typical cases: prototyping a new approach,
quick experiments, throwaway scripts, personal tweaks to a config.

**When NOT to use it**: anything that affects production code in a
non-trivial way, anything security-sensitive, anything that future
you (or a collaborator) would need to audit. The cost of going through
the planner for those cases is very small; the value of having a
trail is large.

**Reverting to standard flow**: if mid-work you decide free mode was
the wrong choice, switch to the planner with `Tab`. The builder
summarizes what was done so far so the planner has context. The
planner can then absorb the in-progress changes into a proper
macro-task or suggest committing/reverting.

## Documentation sync rules

Documentation is **source of truth** and must stay aligned with the code
at all times. Every agent that modifies code is responsible for propagating
the change to the relevant documentation in the same micro-task.

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
| Public service interface (request/response shape)   | `../.docs/contracts/*` — **flag only, do not modify** (see Boundaries) |

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
- The change is then handled from the project root in a separate session
  (planner at project level updates `../.docs/contracts/*` and
  coordinates impacted services)

## Handling significant changes

The default workflow (planner → builder → reviewers) is optimized for
forward progress. Real software also goes through **significant
changes**: specs shifting, architectural patterns being replaced, past
decisions being invalidated, broad refactors. These are not exceptions —
they're part of the lifecycle.

### When to trigger refactor mode

The planner has a dedicated **refactor mode** (see `planner.md`
agent). It is **user-initiated**: the planner does not detect a
refactor on its own. Trigger it explicitly when you know you're about
to change direction:

- "The spec for module X has changed, we need to redesign"
- "Pattern Y isn't working, switch to Z across the service"
- "Decision D-007 was wrong, we need to replace it and fix everything
  downstream"

The planner will then walk you through a structured flow: scoping
(narrow / medium / wide), impact assessment, transitional state setup,
doc updates, refactor macro-task creation.

### Classification

- **Narrow**: one spec, one flow, stays inside a macro-task. Handled
  by re-planning the affected macro-task. No formal refactor mode.
- **Medium**: one or more modules inside this service, invalidates
  past decisions. Enters refactor mode. Produces a dedicated
  `task-NN-refactor-<n>` macro-task.
- **Wide**: service-level architecture, possibly cross-service. Must
  start with a session from the project root to align the cross-service
  story before the service-level refactor proceeds.

### Convention: never delete history

Decisions, task entries, and micro-task logs are **never deleted** —
even when they become obsolete. Instead:

- Decisions use status `superseded by <ref>` with a one-line reason.
  The original text stays.
- Macro-tasks and micro-tasks invalidated by a refactor use status
  `blocked` (will resume) or `skipped` (abandoned). The entry stays
  with a one-line reason pointing to the refactor.
- Completion logs of micro-tasks whose code gets refactored remain
  as-is — they document what was done at that moment in time.

This preserves institutional memory. Anyone reading history in six
months can reconstruct what was decided when and why it changed.

### The refactor task — single source of truth

A refactor is **a macro-task with corrective intent**. Same lifecycle as
any other macro-task (architecture mode → planning → implementation →
wrap-up), with two specializations: naming convention
(`task-NN-refactor-<n>`) and an up-front impact mapping phase that
marks superseded decisions and abandoned work without losing history.

While the refactor task is active, its Plan section (in the macro-task
ledger) is the source of truth for the **target architecture**. The
planner surfaces in every triage that a refactor task is in progress,
so anyone joining mid-refactor knows where to look.

### Docs follow the code during a refactor

`specs.md`, `architecture/architecture.md`, and module docs are **not**
pre-updated when the refactor is started. They keep describing the
current code reality. They are updated by the **wrap-up of the refactor
task**, when the new code is in place — at that point the
architect-service skill in consolidation mode synchronizes the docs to
the new reality.

This means there is a window during which the docs lag behind the
target. That's intentional: the Plan section of the refactor task is
where the target state lives until the refactor lands. Once the
refactor task is `done`, normal doc-code coherence resumes.

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

- `SDD-KEEP: backward compat for v1 API, remove after v2 migration (decisions.md SI-decision-12)`
- `SDD-KEEP: feature-flagged, activated by FEATURE_X (see specs.md)`
- `SDD-KEEP: temporary parallel impl during migration to NewService, remove in task-NN`

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

## File conventions & anti-bloat strategy

This service maintains several incremental files. Each has a defined
growth strategy to prevent bloat. Skills and agent prompts contain the
strict templates; this section is the high-level reference.

| File                                | Owner             | Growth strategy                                      |
| ----------------------------------- | ----------------- | ---------------------------------------------------- |
| `tasks.md`                          | planner           | Compact entries; completed go under `## Completed`   |
| `task-NN-<n>.md` (macro-task ledger)| planner + builder + reviewers | Single file with sections: Plan (immutable, 100-300 lines, planner-owned), Micro-tasks (planner-owned), Emergent decisions (lazy, planner+builder), Execution logs (append-only, builder+reviewers), Wrap-up (planner). Total file size grows with macro-task scope. If a single section grows excessively (e.g. Plan > 500 lines, Execution logs > 2000 lines), the macro-task is probably too big — split it. |
| `.docs/security-issues.md`          | security-reviewer | Open + recently-closed only; archive older closed   |
| `.docs/architecture/architecture.md`| planner           | Split into `modules/<x>.md` if > ~300 lines         |
| `.docs/architecture/modules/<n>.md` | planner           | Created at wrap-up of macro-task introducing the module; updated at subsequent wrap-ups of macro-tasks touching it |
| `.docs/express-log.md`              | builder           | Append-only, one line per fix. Archive older entries manually if needed |

Planner runs lightweight **health checks** during triage and surfaces
suggestions (consolidate, split, archive) — never executes them silently.

## Boundaries

- **Do not** modify project-level docs (`../.docs/*`) directly from this
  level. Cross-service changes must be flagged in the ledger's Emergent decisions section
  — see **Documentation sync rules → Cross-service changes** above.
- **Do not** skip the planner → builder → review chain without explicit
  user instruction.
- **Do** keep incremental files structured and short. Narrative belongs
  in chat, not in committed files.
