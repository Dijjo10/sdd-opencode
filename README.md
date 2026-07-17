# sdd-opencode

> A Spec-Driven Development pattern for agentic coding in
> [OpenCode](https://opencode.ai/). A disciplined workflow built on
> specialized agents, explicit documentation, and atomic tasks —
> designed to keep AI-assisted development auditable and coherent.

---

## Status

**Pattern proposal — v0.2, refined through real-project iteration.**

The methodology, file structure, and agent prompts in this repo are
documented and internally coherent. v0.2 folds in lessons from driving the
pattern through a real multi-service project: most changes turned advisory
steps into **verifiable gates**, moved framework knowledge into **on-demand
skills**, and closed gaps the first version left open. It is still not
validated across *many* projects — an honest, hardened starting point, not a
finished product.

What v0.2 added over v0.1:

- **Adaptive triage / fast-path** — skip the full triage when a macro-task is
  already in progress, going straight to the next micro-task
- **Blocking wrap-up gates** — full test suite + a **CI gate** mirroring the
  workflow yaml + a non-skippable aggregate review + a commit proposal, all
  as exit preconditions (not skippable steps)
- **End-of-session commit gate** — no session that touched files ends without
  proposing a commit
- **Context7 (live docs) mandatory** for planner/builder before writing
  library-specific code — verify, don't guess
- **Third-party APIs as source of truth** — a local `external-<provider>.yaml`
  contract plus a mandatory fetch of the provider's live docs
- **Mockups as source of truth** for frontend services (opt-in)
- **Chronological task-log** decoupled from task numbering
- **Dynamic findings** — a framework-findings workflow that captures recurring
  gotchas and promotes them into on-demand skills

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
5. **Review** is automatic and layered (code quality + security)
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

- [**OpenCode**](https://opencode.ai/) installed and working
- An LLM provider configured in OpenCode with **at least two tiers of
  models available**:
  - A heavy/reasoning model for `planner`, `code-reviewer`,
    `security-reviewer` — prioritize judgment quality
  - A light/fast model for `builder` — prioritize throughput
- The **Context7 MCP** available in OpenCode (the templates enable it in
  `opencode.json`) — agents use it to fetch live library docs before writing
  library-specific code. If you don't want it, remove the `mcp.context7`
  block; the Context7 guidance in the prompts then simply won't fire.
- A monorepo layout works best, but the pattern adapts to single-service
  projects (see "Single-service projects" below)
- Familiarity with OpenCode's agent, skill, and permission model
  (see [OpenCode docs](https://opencode.ai/docs/))

---

## Quick start

### 1. Copy the scaffolding into your project

```
your-project/
├── _project/                          # project workspace (separate OpenCode workspace)
│   ├── AGENTS.md                      # from templates/_project/
│   ├── opencode.json                  # from templates/_project/
│   ├── .opencode/
│   │   ├── agents/
│   │   │   └── principal.md           # the project-level agent
│   │   └── skills/
│   │       ├── analyst/SKILL.md
│   │       └── architect-project/SKILL.md
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
│   │       └── findings-workflow/SKILL.md
│   ├── .docs/
│   └── <source-tree>/
│
├── README.md                          # root README (edited from _project/)
├── docker-compose.yaml                # root compose (edited from _project/)
└── .github/workflows/                 # CI/CD (edited from _project/)
```

**Note on workspaces**: OpenCode loads a single `opencode.json` per
session. To keep project-level and service-level configurations
isolated, each level is its own workspace:

- Run `cd _project && opencode` for project-level work (PDR,
  architecture, contracts, scaffolding)
- Run `cd <service-dir> && opencode` for service-level work
  (implementation, tests, internal architecture)

The two workspaces communicate via files on disk (the cross-service
docs and contracts produced in `_project/.docs/` are read by service
workspaces when relevant).

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
  "agent": {
    "principal":         { "model": "your-heavy-model-id" },
    "planner":           { "model": "your-heavy-model-id" },
    "builder":           { "model": "your-light-model-id" },
    "code-reviewer":     { "model": "your-heavy-model-id" },
    "security-reviewer": { "model": "your-heavy-model-id" }
  }
}
```

Replace `your-heavy-model-id` with a strong reasoning model and
`your-light-model-id` with a fast/cheap model. OpenCode uses the
format `provider/model-id` — run `opencode models` to see what's
available in your installation.

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
| `principal`    | primary | heavy      | PDR, cross-service architecture, contracts, infra, project-wide scaffolding (CI/CD, root Dockerfile, root README) |

**In a service workspace** (when you run `opencode` from a service folder):

| Agent               | Type    | Model tier | Role                                                          |
| ------------------- | ------- | ---------- | ------------------------------------------------------------- |
| `planner`           | primary | heavy      | Triages state, writes service docs, plans tasks, runs wrap-ups|
| `builder`           | primary | light      | Implements one micro-task per session, writes tests, iterates |
| `code-reviewer`     | subagent| heavy      | Reviews diff for code quality, applies fixes directly         |
| `security-reviewer` | subagent| heavy      | Read-only security review, invoked when task is flagged       |

Switch primary agents (planner ↔ builder) with `Tab`. Subagents are
invoked by the builder explicitly when needed.

> **Why two workspaces?** OpenCode loads a single `opencode.json` per
> session — the one in the directory where you launch `opencode`. It
> does not merge nested configs. To keep project-level and
> service-level concerns properly isolated (different agents,
> different models, different context), each level is its own
> workspace. The `_project/` folder is **not a service** — it's the
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
│  ├─ Wrap-up (GATE)  → aggregate review + full tests + CI gate   │
│  │                    + commit — all as exit preconditions       │
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
│  ├─ Implement micro-task + tests (module-scoped)                │
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
| `task-decomposition`             | service | Breaking macro-tasks into atomic micro-tasks  |
| `findings-workflow`              | service | Capturing recurring framework gotchas and promoting them into a findings skill |
| `<stack-name>` (user-provided)   | service | Curated stack-specific conventions (optional) |
| `<framework>-findings` (auto)    | service | Experiential findings for a framework, grown on the job, loaded on demand |

### Documentation structure

The `.docs/` directory holds project knowledge at two levels:

**Project level (`.docs/` at root)** — auto-loaded for project sessions:
- `PDR.md` — requirements
- `architecture.md` — cross-service design
- `infra.md` — environments, deployment
- `contracts/openapi-<ms>.yaml` — inter-service interface definitions
- `contracts/external-<provider>.yaml` — third-party API surfaces you depend
  on (structural source of truth; the provider's live docs are the semantic
  one)
- `mockups/` + `mockups/index.md` — UI prototypes (only for projects with a
  frontend; the source of truth for anything visual)

**Service level (`<service-dir>/.docs/`)** — auto-loaded for service
sessions, partitioned per agent (planner sees more, builder/reviewers
see less):
- `specs.md` — what the service does
- `architecture/architecture.md` + `modules/<n>.md` — internal design
- `infra/server.md` + `infra/ops.md` — runtime and deployment
- `security-issues.md` — service security registry
- `findings/<framework>.md` — append-only raw log of framework gotchas
- `tasks/tasks.md` — macro-tasks index (grouped by state)
- `tasks/task-log.md` — chronological, append-only timeline of task events
  (decoupled from the stable task numbering)
- `tasks/task-NN-<n>.md` — single ledger per macro-task with Plan,
  Micro-tasks, Emergent decisions, Execution logs, Wrap-up sections

Files frequently updated (task ledgers, decisions, findings) are lazy-loaded
— the agent reads them when navigation requires.

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
| Wrap-up (gate) | All micro-tasks of a macro-task done                   | Aggregate review + full tests + CI gate + commit — all as exit preconditions |
| Refactor       | User says "refactor X" / significant pivot             | Impact report + new refactor macro-task         |
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
- **Dual-model setup** (heavy for judgment, light for throughput) —
  the heavy model is used where mistakes are expensive (planning,
  reviewing); the light model is used where iteration is frequent
  (implementation).
- **Architecture-first at the macro-task level** — every macro-task
  begins with a Plan section in its ledger, capturing objective, scope,
  design decisions with alternatives, and approach, before any
  decomposition or implementation.
- **Single ledger per macro-task** — Plan, Micro-tasks, Emergent
  decisions, Execution logs, and Wrap-up live in one file
  (`task-NN-<n>.md`) for fast access and zero risk of file desync.
- **Context partitioning per agent** — `opencode.json` declares
  `instructions` per-agent so the builder doesn't get saturated with
  PDR / contracts / security registries it doesn't need.
- **Adaptive triage / fast-path** — when a macro-task is already in
  progress, the planner skips configuration sync and the full triage and
  goes straight to detailing the next micro-task, to save context budget.
- **Wrap-up is a hard gate, not a checklist** — a macro-task cannot be
  marked `done` unless its exit preconditions hold: the aggregate review
  ran (section in the ledger), the **full test suite is green**, the **CI
  gate** (checks derived from the workflow yaml) is green or CI is
  undefined, and a **commit was proposed**. Two-tier tests still apply:
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
- **Stable task IDs, chronological log** — task numbers never change; a
  separate append-only `task-log.md` records the real execution order, so
  inserting an urgent `task-20` ahead of planned `15–19` doesn't scramble
  history.
- **Findings become skills** — recurring framework/library gotchas are
  logged and, once they recur, promoted into an on-demand
  `<framework>-findings` skill, so the builder stops re-hitting them.
  Lessons route by a 3-way rule: universal → the pattern, project-flow →
  `AGENTS.md`, framework → a findings skill (never all dumped in one place).
- **History is preserved** — decisions, tasks, and execution logs are
  never deleted, only superseded.

---

## Customization

### Adding stack-specific commands to the bash whitelist

The builder ships with a stack-agnostic bash whitelist (`ls`, `cat`,
`git status`, etc.). To avoid being asked confirmation for every
build/test invocation, extend `agent.builder.permission.bash` in your
service's `opencode.json` with your stack's commands. Examples:

**JVM**:
```json
"./gradlew *": "allow",
"./mvnw *": "allow"
```

**Node**:
```json
"npm *": "allow",
"npx *": "allow",
"pnpm *": "allow"
```

**Python**:
```json
"python *": "allow",
"pytest *": "allow",
"poetry *": "allow",
"uv *": "allow"
```

**Go**:
```json
"go *": "allow"
```

Adapt to your actual tooling. Keep destructive commands (`rm`, `sudo`,
`curl`, `wget`) at `ask` or `deny`.

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
GraphQL SDL), update the `instructions` glob in `opencode.json` and
adjust the `architect-project` skill's templates accordingly.

Third-party APIs get their own `external-<provider>.yaml` contract, kept
distinct from the inter-service `openapi-<ms>.yaml` files so it's obvious
which interfaces you own versus depend on.

### Frontend services and mockups

If a service has a UI, drop your prototypes in `_project/.docs/mockups/`
with an `index.md`, and load them into the frontend service's `planner`/
`builder` `instructions`. The pattern treats the HTML mockup as the source
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
3. Set models in the new `opencode.json` if you don't have a global
   config
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

The two-level structure exists for monorepos. For single-service it's
overhead.

### Disabling OpenCode's built-in agents

OpenCode ships with two primary agents: `plan` and `build`. The
templates already disable them in `opencode.json` (`"disable": true`)
to avoid Tab-cycle clutter with the SDD planner/builder. If you want
them back, remove the disable entries.

---

## Repository layout

```
sdd-opencode/
├── README.md                       # you are here
├── LICENSE
├── templates/
│   ├── _project/                   # template for the project workspace
│   │   ├── AGENTS.md
│   │   ├── opencode.json
│   │   └── .opencode/
│   │       ├── agents/
│   │       │   └── principal.md
│   │       └── skills/
│   │           ├── analyst/SKILL.md
│   │           └── architect-project/SKILL.md
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

**The builder is asking confirmation for every build/test command.**
Extend the bash whitelist in `agent.builder.permission.bash` of your
service's `opencode.json`. See "Customization" above for examples.

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
The templates enable the Context7 MCP in `opencode.json`. If it isn't
available in your OpenCode install, either add it, or remove the
`mcp.context7` block — the prompts will still work, they just won't fetch
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
primitives (skills, primary/subagent model, per-agent `instructions`).
