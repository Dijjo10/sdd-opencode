# sdd-opencode

> A Spec-Driven Development pattern for agentic coding in
> [OpenCode](https://opencode.ai/). A disciplined workflow built on
> specialized agents, explicit documentation, and atomic tasks —
> designed to keep AI-assisted development auditable and coherent.

---

## Status

**Pattern proposal — v0.3, for OpenCode 2.x.** Refined on two real
multi-service projects.

> Using **OpenCode 1.x**? Use the [`v1-legacy`](../../tree/v1-legacy) tag.
> v0.3 uses the native OpenCode 2 config format and does not run on 1.x.
> To move an existing project, see [MIGRATION.md](./MIGRATION.md).

What v0.3 adds over v0.2:

- **Native OpenCode 2 config** — `agents`, ordered `permissions`,
  `mcp.servers`
- **Permissive by default** — agents work without interruptions; only
  actions that leave the machine or destroy history ask (push, merge,
  release, `reset --hard`, `clean`, `sudo`); force-push is denied
- **`sdd.config.yaml`** — one small file at the monorepo root for CI, git
  and versioning settings, read by every agent
- **Optional CI** — with no CI the CI gate does not exist; a per-micro-task
  quality gate (lint/format/type-check/tests) and the full suite at
  wrap-up always apply
- **Git workflow** — branch per macro-task (or per version), pull request,
  CI checks, merge on approval, tag at version close; works with GitHub,
  any other remote, or no remote
- **Task IDs with types and versions** — `03-feat-auth`, `12b-fix-token`,
  grouped in `v0.2/` folders
- **Smaller task files** — ledger and execution log split, size budgets,
  compaction at every wrap-up (one-line indexes, ≤ 30-line Summary)
- **Leaner sessions** — rare procedures (wrap-up, express, free, refactor)
  moved into on-demand skills; ~25% less fixed context per session
- Lessons from the field: risk notes with plan B, tests always in the same
  micro-task, schema-first data shapes, "check the upstream guarantee
  first" for security findings

Feedback, issues, and pull requests are very welcome. This pattern is
meant to evolve based on real use — if you adopt it and hit rough
edges, please open an issue describing what broke and what helped.

---

## What is SDD and why this repo

**Spec-Driven Development (SDD)** is a methodology where AI-assisted
coding is governed by well-defined specifications, roles, and decision
lifecycles rather than ad-hoc prompting. Instead of "tell the agent
what to do and hope for the best", the agent operates inside a
structured workflow:

1. **Requirements** are elicited and written down explicitly (PDR)
2. **Architecture** is designed before code — at project level and at
   service level
3. **Tasks** are decomposed into atomic, testable micro-tasks
4. **Implementation** happens one micro-task at a time, with a fresh
   session each
5. **Review** is layered: security review automatic where it matters,
   code review on demand per micro-task and mandatory at wrap-up
6. **Documentation** stays synchronized with the code at every step

The aim is reproducibility and auditability. Every decision leaves a
trail. Every architectural change has a place to live. The agent doesn't
drift, because its context is explicit and its boundaries are codified.

This repo provides the **concrete scaffolding** to do SDD in OpenCode:
agent definitions, skill files, context files, and configuration
templates. It's framework-agnostic and model-agnostic — you plug in
your stack and your LLM provider.

---

## Prerequisites

- [**OpenCode 2.x**](https://opencode.ai/) installed and working
  (`opencode --version` prints 2.x)
- An LLM provider configured in OpenCode. Two tiers of models work best:
  - a heavy/reasoning model for `code-reviewer` and `security-reviewer`
    (the judgment-heavy gates) and for `principal`
  - a light/fast model is enough for `planner` and `builder`
    (high-frequency triage, detailing, implementation)
- The **Context7 MCP** — the templates declare it in `opencode.json`
  (`mcp.servers.context7`). Agents use it to fetch live library docs
  before writing library-specific code. If you don't want it, remove the
  block; the Context7 guidance in the prompts then simply won't fire.
- Optional: the [GitHub CLI](https://cli.github.com/) (`gh`) when
  `git.remote: github` — used for pull requests, CI checks and merges
- A monorepo layout works best, but the pattern adapts to single-service
  projects (see "Single-service projects" below)
- Familiarity with OpenCode's agent, skill, and permission model
  (see [OpenCode docs](https://opencode.ai/docs/))

---

## Quick start

### 1. Copy the scaffolding into your project

```
your-project/
├── sdd.config.yaml                    # from templates/ — CI, git, versioning settings
├── _project/                          # project workspace (separate OpenCode workspace)
│   ├── AGENTS.md                      # from templates/_project/
│   ├── opencode.json                  # from templates/_project/
│   ├── .opencode/
│   │   ├── agents/
│   │   │   └── principal.md           # the project-level agent
│   │   └── skills/
│   │       ├── analyst/SKILL.md
│   │       ├── architect-project/SKILL.md
│   │       └── git-workflow/SKILL.md
│   └── .docs/                         # to be populated by the analyst
│       └── contracts/                 # to be populated during architecture
│
├── <service-dir>/                     # one or more services (each is its own workspace)
│   ├── AGENTS.md                      # from templates/service/
│   ├── opencode.json                  # from templates/service/
│   ├── .opencode/
│   │   ├── agents/
│   │   │   ├── planner.md
│   │   │   ├── builder.md
│   │   │   ├── code-reviewer.md
│   │   │   └── security-reviewer.md
│   │   └── skills/
│   │       ├── architect-service/SKILL.md
│   │       ├── task-decomposition/SKILL.md
│   │       ├── wrap-up/SKILL.md
│   │       ├── git-workflow/SKILL.md
│   │       ├── express-mode/SKILL.md
│   │       ├── free-mode/SKILL.md
│   │       ├── refactor-mode/SKILL.md
│   │       └── findings-workflow/SKILL.md
│   ├── .docs/
│   └── <source-tree>/
│
├── README.md                          # root README (edited from _project/)
├── docker-compose.yaml                # root compose (edited from _project/)
└── .github/workflows/                 # CI/CD (edited from _project/)
```

**Note on workspaces**: the project level and each service are
separate OpenCode workspaces, side by side (no `opencode.json` at the
monorepo root), so their configs and agents never mix:

- Run `cd _project && opencode` for project-level work (PDR,
  architecture, contracts, scaffolding)
- Run `cd <service-dir> && opencode` for service-level work
  (implementation, tests, internal architecture)

The two workspaces communicate via files on disk (the cross-service
docs and contracts produced in `_project/.docs/` are read by service
workspaces when relevant).

`sdd.config.yaml` can stay as copied: at the first project session the
`principal` asks you the few values it holds (CI provider, git remote and
flow, branch per task or per version, versioning on/off). If the file is
missing, agents assume no CI, no remote and no versioning.

### 2. Set your models in your global OpenCode config

The `opencode.json` files in this template **do not declare models**.
Models are configured **once** in your personal global OpenCode config
at `~/.config/opencode/opencode.json` (or `%APPDATA%\opencode\opencode.json`
on Windows). This way:

- Your model choice stays **out of the repo** — no risk of pushing
  personal config or accidentally committing model preferences
- Different team members can use different models on the same
  project without touching the shared `opencode.json`
- One change in the global config applies to all your SDD projects

Add this to your global `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agents": {
    "principal":         { "model": "provider/heavy-model" },
    "planner":           { "model": "provider/light-model" },
    "builder":           { "model": "provider/light-model" },
    "code-reviewer":     { "model": "provider/heavy-model" },
    "security-reviewer": { "model": "provider/heavy-model" }
  }
}
```

OpenCode 2 uses `provider/model` with an optional variant after `#`
(e.g. `anthropic/claude-sonnet-5-5#high`) — run `opencode models` to see
what's available. Want maximum judgment everywhere? Use the heavy model
for `planner` too; the split above is the one that worked best in field
use (reviews carry the quality gates, planner and builder iterate often).

OpenCode merges the global config with the project's `opencode.json`,
so models from your global config apply automatically to all the
agents in this pattern.

**Other placeholders** in `.md` files (service name, stack name, etc.)
are handled **automatically** by the planner/principal agent at the
first session via the **configuration sync** procedure. You don't
need to find-and-replace them manually.

### 3. Launch

For project-level work (PDR, architecture, contracts, scaffolding):

```bash
cd your-project/_project
opencode
```

OpenCode starts in the `principal` agent by default. Say hi or type
`go` — the agent runs configuration sync (substituting any unresolved
placeholders), then triages project state. For a fresh project, it
will invoke the `analyst` skill to build the PDR.

For service-level work (after project-level architecture is in place):

```bash
cd your-project/<service-dir>
opencode
```

The service workspace starts in the `planner` agent by default.

---

## Pattern overview

### Agents

The pattern uses **two kinds of OpenCode workspaces** — project-level
and service-level. Each is independent: its own `opencode.json`, its
own agents, its own context. They communicate via files on disk
(cross-service docs and contracts produced in the project workspace
are read by service workspaces when relevant).

**In the project workspace** (when you run `opencode` from `_project/`):

| Agent          | Type    | Model tier | Role                                                                                |
| -------------- | ------- | ---------- | ----------------------------------------------------------------------------------- |
| `principal`    | primary | heavy      | `sdd.config.yaml`, version close, PDR, cross-service architecture, contracts, infra, project-wide scaffolding (CI/CD, root Dockerfile, root README) |

**In a service workspace** (when you run `opencode` from a service folder):

| Agent               | Type    | Model tier | Role                                                          |
| ------------------- | ------- | ---------- | ------------------------------------------------------------- |
| `planner`           | primary | light/heavy| Triages state, writes service docs, plans tasks, runs wrap-ups|
| `builder`           | primary | light      | Implements one micro-task per session, writes tests, iterates |
| `code-reviewer`     | subagent| heavy      | Reviews diff for code quality, applies fixes directly         |
| `security-reviewer` | subagent| heavy      | Read-only security review, invoked when task is flagged       |

Switch primary agents (planner ↔ builder) with `Tab`. The builder invokes
`@security-reviewer` automatically for security-relevant micro-tasks and
`@code-reviewer` on request; the planner invokes both in aggregate at
wrap-up.

> **Why two workspaces?** OpenCode reads the `opencode.json` and the
> `.opencode/agents/` folders found from the directory where you launch
> it up the tree, into one agent namespace. Putting project-level and
> service-level agents on the same path would mix them. So each level
> is its own workspace, side by side, with no `opencode.json` at the
> monorepo root. The `_project/` folder is **not a service** — it's the
> project workspace, sitting alongside the service folders.

### Workflow

The diagram below shows the workflow **inside a service workspace**.
The project workspace (`_project/`) is simpler: a single `principal`
agent that handles requirements, architecture, contracts, and
scaffolding — typically used at project setup, when adding a new
service, or when a cross-service architectural change is needed.

```
┌─────────────────────────────────────────────────────────────────┐
│  Session start → planner (default agent)                         │
│  │                                                               │
│  ├─ Fast-path: macro-task in progress? → jump to detailing       │
│  │             (skip config sync / triage)                       │
│  ├─ Else full triage: config sync (setup-phase) → assess state   │
│  │                                                               │
│  ├─ Setup mode      → architect-service skill → specs + arch    │
│  │                    (+ mandatory .gitignore / bootstrap)       │
│  ├─ Brownfield mode → architect-service (reverse-engineering)   │
│  ├─ Architecture    → architect-service skill → Plan section    │
│  ├─ Planning mode   → task-decomposition → Micro-tasks + log    │
│  ├─ Detailing mode  → plan next micro-task in chat (+ mockup)   │
│  ├─ Blocked resolve → verify & unblock blocked micro-tasks      │
│  ├─ Wrap-up (GATE)  → wrap-up skill: aggregate reviews + full   │
│  │                    tests + CI gate (if CI) + compaction +     │
│  │                    commit + integration (push/PR/merge)       │
│  ├─ Refactor mode   → scope change, impact map, refactor task   │
│  ├─ Express mode    → planner-gated trivial fix (typo, bump)    │
│  │                                                               │
│  │  Context7 + stack skill are mandatory before planning        │
│  │  library-specific code. End-of-session commit gate applies.  │
│  └─ Handoff to builder (Tab)                                    │
│                                                                  │
│  builder                                                         │
│  │                                                               │
│  ├─ Context7 + stack skill + <framework>-findings before coding │
│  ├─ Implement micro-task + tests (module-scoped) + quality gate  │
│  ├─ Third-party API? → fetch live docs first (source of truth)  │
│  ├─ Record framework gotchas → findings-workflow                │
│  ├─ Write completion log + doc-sync declaration                  │
│  ├─ Conditional: @security-reviewer if security-relevant        │
│  ├─ On-demand: @code-reviewer when user asks                    │
│  ├─ Free mode: direct-to-builder, zero ceremony, untracked      │
│  └─ Session ends. User opens fresh session for next micro-task   │
└─────────────────────────────────────────────────────────────────┘
```

### Skills

Skills encode **methodology**, loaded on-demand by agents:

| Skill                            | Level   | When used                                     |
| -------------------------------- | ------- | --------------------------------------------- |
| `analyst`                        | root    | Eliciting project requirements (PDR)          |
| `architect-project`              | root    | Designing cross-service architecture + third-party contracts |
| `architect-service`              | service | Designing internal architecture (3 modes) + brownfield reverse-engineering |
| `task-decomposition`             | service | Task IDs and layout, decomposition, ledger + log format, size budgets, compaction |
| `wrap-up`                        | service | The planner's wrap-up procedure (loaded at every wrap-up) |
| `git-workflow`                   | both    | Branch at task start, PR/CI/merge at wrap-up, version close and tag, branch protection |
| `express-mode` / `free-mode`     | service | Trivial tracked changes / untracked changes   |
| `refactor-mode`                  | service | Changes of direction: scoping, impact mapping, `ref` task |
| `findings-workflow`              | service | Capturing recurring framework gotchas and promoting them into a findings skill |
| `<stack-name>` (user-provided)   | service | Curated stack-specific conventions (optional) |
| `<framework>-findings` (auto)    | service | Experiential findings for a framework, grown on the job, loaded on demand |

### Documentation structure

The `.docs/` directory holds project knowledge at two levels:

**Project level (`_project/.docs/`)** — read on demand by `principal`
(nothing is eager-loaded in the project workspace):
- `PDR.md` — requirements
- `architecture.md` — cross-service design
- `infra.md` — environments, deployment
- `contracts/openapi-<ms>.yaml` — inter-service interface definitions
- `contracts/external-<provider>.yaml` — third-party API surfaces you depend
  on (structural source of truth; the provider's live docs are the semantic
  one)
- `mockups/` + `mockups/index.md` — UI prototypes (only for projects with a
  frontend; the source of truth for anything visual)
- `source/` — reference material you provide (read-only for agents)

**Service level (`<service-dir>/.docs/`)** — only `specs.md` is
eager-loaded; each agent has a short list of **mandatory reads** (see
"Context partitioning" in the service `AGENTS.md`):
- `specs.md` — what the service does
- `architecture/architecture.md` + `modules/<n>.md` — internal design
- `infra/server.md` + `infra/ops.md` — runtime and deployment
- `security-issues.md` — service security registry
- `findings/<framework>.md` — append-only raw log of framework gotchas
- `tasks/` — the task files:

```
tasks/
├── tasks.md                 # index, one line per macro-task
├── task-log.md              # timeline, one line per event
├── v0.2/                    # version folder (when versioning is on)
│   ├── 03-feat-auth.md      # ledger: Summary, Plan, Micro-tasks, ED, Wrap-up
│   └── 03-feat-auth.log.md  # execution log — read only at wrap-up
└── backlog/
```

Macro-task IDs are `<NN>[letter]-<type>-<slug>` with types `feat`, `fix`,
`ref`, `chore`, `spike`; micro-tasks are `mt-NN`. IDs never change — a task
moved to another version keeps its ID.

### Operating modes

The planner's behavior is shaped by what it detects at triage. Each mode
has a clear purpose:

| Mode           | Trigger                                                | Output                                          |
| -------------- | ------------------------------------------------------ | ----------------------------------------------- |
| Fast-path      | A macro-task is already `in-progress` at session start | Skip triage, go straight to detailing next MT   |
| Setup          | Foundational docs missing                              | `specs.md` + `architecture.md` + `.gitignore`   |
| Brownfield     | User says "modalità adozione" / "brownfield"           | Baseline docs from existing code                |
| Architecture   | New macro-task, no Plan section yet                    | Plan section of macro-task ledger               |
| Planning       | Plan approved, no Micro-tasks section yet              | Micro-tasks section + `task-log` entry          |
| Detailing      | Micro-task ready to start                              | Implementation plan in chat (not on disk)       |
| Wrap-up (gate) | All micro-tasks of a macro-task done                   | Aggregate reviews + full tests + CI gate (if CI) + compaction + commit + integration |
| Refactor       | User says "refactor X" / significant pivot             | Impact report + new `ref` macro-task            |
| Express        | User says "quick fix" / "express", planner-gated       | Implementation + one line in `express-log.md`   |

The **builder** has its own modes when activated:

| Mode    | When                                                            | Discipline                              |
| ------- | --------------------------------------------------------------- | --------------------------------------- |
| Normal  | Plan from planner is in chat context                            | Full SDD: doc-sync, completion log, etc.|
| Express | Planner announced "express mode confirmed"                      | Trivial change + express-log entry      |
| Free    | User opens session, switches directly to builder, no plan       | Zero ceremony, untracked, raw OpenCode  |

### Key design choices

Some non-obvious decisions baked into the pattern:

- **One session = one micro-task** (hard rule). No silent chaining —
  every micro-task gets a fresh context.
- **Two model tiers** (heavy for judgment, light for throughput) —
  the heavy model carries the review gates; the light one iterates
  (triage, detailing, implementation). Models live in your global
  config, never in the repo.
- **Permissive by default, gated where it matters** — no prompts for
  edits, builds, tests or commits; push, merge, release and destructive
  git commands ask; force-push is denied. Server-side branch protection
  is the real safety net.
- **Architecture-first at the macro-task level** — every macro-task
  begins with a Plan section in its ledger, capturing objective, scope,
  design decisions with alternatives, and approach, before any
  decomposition or implementation.
- **Ledger + execution log per macro-task** — intent and state
  (`<id>.md`) are separated from execution details (`<id>.log.md`), so
  the file read at every detailing stays small. Size budgets and a
  compaction step at wrap-up keep indexes at one line per task.
- **Context partitioning per agent** — only `specs.md` is eager-loaded
  (OpenCode has no per-agent `instructions`); each agent has a short
  list of mandatory reads, and rare procedures are skills loaded on
  demand.
- **Adaptive triage / fast-path** — when a macro-task is already in
  progress, the planner skips configuration sync and the full triage and
  goes straight to detailing the next micro-task, to save context budget.
- **Wrap-up is a hard gate, not a checklist** — a macro-task cannot be
  marked `done` unless its exit preconditions hold: the aggregate review
  ran (verdict in the ledger), the **full test suite is green**, the **CI
  gate** (checks derived from the pipeline definition) is green when CI
  is enabled, and a **commit was proposed**. A pull request is never
  merged on red CI. Two-tier tests still apply:
  module-scoped during each micro-task, full suite at wrap-up.
- **End-of-session commit gate** — no session that modified files ends
  without proposing a commit; enforced at every agent, especially after a
  Plan is approved and at wrap-up.
- **Live docs over memory (Context7)** — the planner and builder must
  consult Context7 before planning/writing library-specific code. Verify
  the current API, don't guess from stale knowledge.
- **Code review on-demand, security review automatic** — code-reviewer
  is invoked only when the user asks (or at wrap-up for aggregate
  review). Security-reviewer fires automatically when a micro-task is
  flagged `security-relevant: true`.
- **Documentation sync mandatory** — builder declares updates,
  code-reviewer verifies (when invoked), planner cross-checks at
  wrap-up.
- **Cross-service changes stop the service-level session** — when a
  change requires touching an inter-service contract, it must be
  handled from the `_project/` workspace.
- **Brownfield-friendly** — adoption on existing codebase doesn't
  require rewriting docs. The planner has a dedicated brownfield mode
  that produces a baseline; module docs accumulate lazily.
- **Significant changes have a dedicated flow** — refactor mode in 3
  phases (scoping, impact mapping, create refactor task). The refactor
  task's Plan section is the source of truth for target architecture.
- **Third-party APIs are a tracked source of truth** — a dependency on an
  external provider gets a local `external-<provider>.yaml` contract
  (structural truth), and agents must fetch the provider's live docs before
  coding against it (semantic truth). Working from memory on an external
  surface is a known source of silent bugs.
- **Mockups win for UI (frontend, opt-in)** — for services with a frontend,
  the HTML mockup is the highest-fidelity source of truth; the builder reads
  it before writing a screen and copies its copy/spacing/sizes verbatim.
- **Stable task IDs, chronological log** — task IDs never change; a
  separate append-only `task-log.md` records the real execution order, so
  starting `20-fix-...` ahead of planned `15–19` doesn't scramble
  history.
- **Git is part of the workflow** — branch at macro-task start, explicit-
  path commits per micro-task (several services can share a checkout),
  integration at wrap-up, tag at version close — all driven by
  `sdd.config.yaml`.
- **Findings become skills** — recurring framework/library gotchas are
  logged and, once they recur, promoted into an on-demand
  `<framework>-findings` skill, so the builder stops re-hitting them.
  Lessons route by a 3-way rule: universal → the pattern, project-flow →
  `AGENTS.md`, framework → a findings skill (never all dumped in one place).
- **History is preserved** — decisions, tasks, and execution logs are
  never deleted, only superseded.

---

## Customization

### Tightening or loosening permissions

OpenCode 2 allows everything by default; the templates add an ordered
`permissions` array where **the last matching rule wins**. The shipped
policy:

```json
"permissions": [
  { "action": "shell", "resource": "*git push*",          "effect": "ask"  },
  { "action": "shell", "resource": "*git merge *",        "effect": "ask"  },
  { "action": "shell", "resource": "*gh pr merge*",       "effect": "ask"  },
  { "action": "shell", "resource": "*gh release*",        "effect": "ask"  },
  { "action": "shell", "resource": "*git reset --hard*",  "effect": "ask"  },
  { "action": "shell", "resource": "*git clean*",         "effect": "ask"  },
  { "action": "shell", "resource": "*sudo *",             "effect": "ask"  },
  { "action": "shell", "resource": "*git push*--force*",  "effect": "deny" },
  { "action": "shell", "resource": "*git push*-f *",      "effect": "deny" }
]
```

The leading `*` also catches chained commands (`cd x && git push`). Agent
rules (`agents.<id>.permissions`) are appended after the global ones:
reviewers can never commit, push, reset or merge; the security reviewer
can edit only under `.docs/`. Add rules at the end of the array to
tighten; remove them to loosen. Reading `.env` files asks by default
(OpenCode's base policy) — keep it that way.

> **Unattended runs.** `opencode run --auto` approves every `ask` that is
> not explicitly denied. If you automate the loop, inject `deny` rules for
> what must never happen unattended (push, merge, release) — for example
> through `OPENCODE_CONFIG_CONTENT`.

### Adding a stack-specific skill

For convention-heavy stacks (Spring Boot, Django, Next.js, FastAPI),
you can add an optional skill at `<service-dir>/.opencode/skills/<stack-name>/SKILL.md`
that describes:

- Project layout conventions
- Layering conventions (controllers / services / repositories)
- Naming patterns
- Testing patterns
- Error-handling patterns
- Configuration patterns
- Common idioms to follow / anti-patterns to avoid

The builder loads it autonomously when relevant code is being touched
(based on the skill's description). After you create the skill folder,
the planner's configuration sync will substitute `<stack-name>` in
`AGENTS.md` references at first session.

A minimal skill template:

````markdown
---
description: Stack-specific conventions for <framework>. Loaded by the builder when implementing or reviewing <framework> code, to ensure consistency with project conventions.
---

# <Framework> Conventions

## Project layout
<directory structure conventions>

## Layering
<your layering rules>

## Naming
<your naming patterns>

## Testing
<test framework, conventions, fixtures>

## Error handling
<patterns for exceptions, error responses>

## Configuration
<how config is structured, where it lives>

## Common patterns
- <pattern 1 with example>
- <pattern 2 with example>
````

### Contract format customization

The pattern assumes contracts in `.docs/contracts/*.yaml` (OpenAPI by
default). If you use a different format (gRPC `.proto`, AsyncAPI,
GraphQL SDL), adjust the `architect-project` skill's templates and the
contract rows of the **Documentation map** in the `AGENTS.md` files.

Third-party APIs get their own `external-<provider>.yaml` contract, kept
distinct from the inter-service `openapi-<ms>.yaml` files so it's obvious
which interfaces you own versus depend on.

### Frontend services and mockups

If a service has a UI, drop your prototypes in `_project/.docs/mockups/`
with an `index.md`, and list them among the frontend service's mandatory
reads (**Context partitioning** in its `AGENTS.md`). The pattern treats the HTML mockup as the source
of truth for screens; the "UI work" section in the service `AGENTS.md` is
marked frontend-only and is ignored by headless services.

### Framework findings (experiential knowledge)

The `findings-workflow` skill defines how the builder captures recurring
framework/library gotchas: they land in `.docs/findings/<framework>.md`,
and once one recurs it is promoted into a `<framework>-findings` skill that
loads on demand next to your curated `<stack-name>` skill. The curated skill
plus Context7 always win on conflict — findings are experiential and can
age. Consolidate a findings skill periodically to avoid bloat.

### Bootstrapping a new service in an existing monorepo

When adding a new service to a project that's already using SDD:

1. Copy the `templates/service/*` scaffolding into the new service folder
2. The planner at first session in that folder will run configuration
   sync, detect the new service name from the folder, and substitute
   `<service-name>` everywhere
3. Models come from your global config; nothing to set per service
4. Update the root `architecture.md` to mention the new service and its
   relationships
5. Add contracts in `.docs/contracts/` if the service exposes an API

### Single-service (non-monorepo) projects

The pattern works for single-service projects with minor adjustments:

- Skip the `<service-dir>/` layer — put everything at root
- Merge the project-level and service-level `AGENTS.md` into one
- Drop the `architect-project` skill (you don't have cross-service
  architecture)
- Use a single `opencode.json` with both planner and builder declared
- Put `sdd.config.yaml` at the root next to it (paths `../sdd.config.yaml`
  in the prompts become `./sdd.config.yaml`)

The two-level structure exists for monorepos. For single-service it's
overhead.

### Disabling OpenCode's built-in agents

OpenCode ships with two primary agents: `plan` and `build`. The
templates already disable them in `opencode.json` (`"disabled": true`)
to avoid Tab-cycle clutter with the SDD planner/builder. If you want
them back, remove the disable entries.

---

## Repository layout

```
sdd-opencode/
├── README.md                       # you are here
├── LICENSE
├── MIGRATION.md                    # moving a project from v0.2 (OpenCode 1.x)
├── templates/
│   ├── sdd.config.yaml             # pattern settings, copied to the monorepo root
│   ├── _project/                   # template for the project workspace
│   │   ├── AGENTS.md
│   │   ├── opencode.json
│   │   └── .opencode/
│   │       ├── agents/
│   │       │   └── principal.md
│   │       └── skills/
│   │           ├── analyst/SKILL.md
│   │           ├── architect-project/SKILL.md
│   │           └── git-workflow/SKILL.md
│   └── service/                    # template for a service workspace
│       ├── AGENTS.md
│       ├── opencode.json
│       └── .opencode/
│           ├── agents/
│           │   ├── planner.md
│           │   ├── builder.md
│           │   ├── code-reviewer.md
│           │   └── security-reviewer.md
│           └── skills/
│               ├── architect-service/SKILL.md
│               ├── task-decomposition/SKILL.md
│               ├── wrap-up/SKILL.md
│               ├── git-workflow/SKILL.md
│               ├── express-mode/SKILL.md
│               ├── free-mode/SKILL.md
│               ├── refactor-mode/SKILL.md
│               └── findings-workflow/SKILL.md
└── docs/
    └── examples/                   # (optional, future) example projects
```

---

## Troubleshooting

A few situations users may hit:

**The planner keeps asking me to confirm placeholder substitutions
every session.**
The configuration sync should be silent once values are stable. If it
keeps firing, check that the substitutions were actually applied to
disk (the planner may have proposed but not written if the user didn't
confirm explicitly).

**An agent asks confirmation for a command I consider safe.**
With the v0.3 templates only push/merge/release/destructive git and
`sudo` ask. If something else asks, check your global config for extra
rules (the last matching rule wins). See "Tightening or loosening
permissions" above.

**Project rules are ignored after upgrading OpenCode.**
OpenCode 2 reads only `AGENTS.md`, never `CLAUDE.md`. The templates use
`AGENTS.md` only; move any rule you kept in a `CLAUDE.md` into it.

**A micro-task is much bigger than I expected.**
Stop, switch back to the planner, ask to revise the Micro-tasks section
of the ledger. Atomicity rule: one micro-task should fit in one focused
session.

**Wrap-up is blocked because full test suite fails.**
Either fix the regression in a follow-up micro-task, or mark the
failing test as legitimately broken (rare — needs explicit reasoning
in the wrap-up summary). Don't bypass the gate silently.

**I want to make a change but the planner says it's not eligible for
express mode.**
Either (a) the change is genuinely non-trivial — go through normal
flow, or (b) you want to escape SDD entirely for this change — use
free mode (Tab directly to builder, give the instruction; the builder
will detect free mode and ask explicit confirmation).

**The agents keep saying they can't reach Context7.**
The templates declare Context7 as a remote MCP server in `opencode.json`
(`mcp.servers.context7`). If it is unreachable (network, rate limit),
add an API key header (`"headers": { "CONTEXT7_API_KEY":
"{env:CONTEXT7_API_KEY}" }`) or remove the block — the prompts will still work, they just won't fetch
live docs (falling back to the curated stack skill and your own review).

**The builder isn't recording findings.**
Findings are builder-owned and, like all gates here, are strong instructions
rather than a technical enforcement. If they're being skipped, prompt the
builder to follow the `findings-workflow` when it hits a framework issue; the
mechanism is worth verifying on your first real project.

**The pattern feels heavy for a tiny prototype.**
It is. SDD is for serious projects. For prototypes and throwaway
scripts, use raw OpenCode without this pattern.

---

## Comparison with other approaches

A few words on how this pattern relates to alternatives:

**vs. raw prompting / vibe coding**: you gain reproducibility,
auditability, and discipline at the cost of upfront structure. Fitting
for serious projects, overkill for throwaway scripts.

**vs. fully autonomous agents**: this pattern deliberately keeps the
user in the loop at every micro-task boundary. Slower than "agent goes
off and builds a feature", but far easier to correct course and to
trust.

**vs. prompt libraries / chain-of-thought templates**: those work at
the prompt level. This works at the project level — it shapes how the
entire repository is organized.

---

## Contributing

Issues and pull requests are welcome, especially:

- Reports from adopting the pattern on a real project (what worked,
  what didn't)
- Stack-specific skills (Spring Boot, Django, Next.js, FastAPI, etc.)
  contributed under `docs/examples/`
- Improvements to agent prompts around edge cases
- Translations of this README

Before substantial changes, open an issue first to discuss — the goal
is coherent evolution, not feature creep.

---

## License

MIT. See [LICENSE](./LICENSE) for details.

---

## Acknowledgments

This pattern emerged from iterative design sessions on SDD for agentic
coding. The core ideas — spec-first development, bounded context per
micro-task, decision lifecycle, documentation as source of truth —
are not new. What's specific here is how they map to OpenCode's
primitives (skills, primary/subagent agents, ordered permissions,
mandatory reads instead of eager context).
