# sdd-opencode

> A Spec-Driven Development pattern for agentic coding in
> [OpenCode](https://opencode.ai/). A disciplined workflow built on
> specialized agents, explicit documentation, and atomic tasks —
> designed to keep AI-assisted development auditable and coherent.

---

## Status

**Pattern proposal — v0.1, not yet battle-tested.**

The methodology, file structure, and agent prompts in this repo are
documented and internally coherent, but have not been validated across
multiple real-world projects. This is an honest starting point, not a
finished product.

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
- A monorepo layout works best, but the pattern adapts to single-service
  projects (see "Single-service projects" below)
- Familiarity with OpenCode's agent, skill, and permission model
  (see [OpenCode docs](https://opencode.ai/docs/))

---

## Quick start

### 1. Copy the scaffolding into your project

```
your-project/
├── AGENTS.md                    # from this repo (root AGENTS.md)
├── opencode.json                # from this repo (root opencode.json)
├── .opencode/
│   ├── agents/
│   │   ├── planner-root.md      # from this repo (root planner)
│   │   └── builder-root.md      # from this repo (root builder, scaffolding only)
│   └── skills/
│       ├── analyst/SKILL.md
│       └── architect-project/SKILL.md
├── .docs/                       # to be populated by the analyst
│   └── contracts/               # to be populated during architecture
└── <service-dir>/               # one or more services
    ├── AGENTS.md                # from this repo (service AGENTS.md)
    ├── opencode.json            # from this repo (service opencode.json)
    ├── .opencode/
    │   ├── agents/
    │   │   ├── planner.md
    │   │   ├── builder.md
    │   │   ├── code-reviewer.md
    │   │   └── security-reviewer.md
    │   └── skills/
    │       ├── architect-service/SKILL.md
    │       └── task-decomposition/SKILL.md
    └── .docs/
```

### 2. Set your models in `opencode.json`

The **only manual customization** required before launching is setting
the model identifiers in each `opencode.json` (root and each service).
Open the file and replace `<your-heavy-model>` and `<your-light-model>`
with your chosen model IDs:

```json
{
  "agent": {
    "planner": { "model": "your-heavy-model-id", ... },
    "builder": { "model": "your-light-model-id", ... },
    "code-reviewer": { "model": "your-heavy-model-id", ... },
    "security-reviewer": { "model": "your-heavy-model-id", ... }
  }
}
```

OpenCode uses the format `provider/model-id`. Run `opencode models`
to see what's available in your configuration.

**All other placeholders** in `.md` files (service name, stack name,
etc.) are handled **automatically** by the planner agent at the first
session via its **configuration sync** procedure. You don't need to
find-and-replace them manually.

If your heavy/light model choice is stable across all your projects,
you can declare it once in your global `~/.config/opencode/opencode.json`
and remove the `model` field from per-project files (it'll fall back
to global).

### 3. Launch

From project root:

```bash
cd your-project
opencode
```

OpenCode starts in the `planner` agent by default. Say hi or type `go`
— the planner runs configuration sync (substituting any unresolved
placeholders), then triages project state. For a fresh project, it
will invoke the `analyst` skill to build the PDR.

For service-level work (after project-level architecture is in place):

```bash
cd your-project/<service-dir>
opencode
```

---

## Pattern overview

### Agents

The pattern uses two layers of agents — **project-level** at the
monorepo root, **service-level** inside each service folder. Names
are intentionally distinct (`-root` suffix at root) to avoid
namespace collisions when OpenCode merges agent definitions across
config layers.

**At project root** (when you run `opencode` from monorepo root):

| Agent          | Type    | Model tier | Role                                                              |
| -------------- | ------- | ---------- | ----------------------------------------------------------------- |
| `planner-root` | primary | heavy      | PDR, cross-service architecture, infra, contracts                 |
| `builder-root` | primary | light      | Project-level scaffolding only (CI/CD, root Dockerfile, etc.)     |

**Inside a service** (when you run `opencode` from a service folder):

| Agent               | Type    | Model tier | Role                                                          |
| ------------------- | ------- | ---------- | ------------------------------------------------------------- |
| `planner`           | primary | heavy      | Triages state, writes service docs, plans tasks, runs wrap-ups|
| `builder`           | primary | light      | Implements one micro-task per session, writes tests, iterates |
| `code-reviewer`     | subagent| heavy      | Reviews diff for code quality, applies fixes directly         |
| `security-reviewer` | subagent| heavy      | Read-only security review, invoked when task is flagged       |

Switch primary agents (planner ↔ builder) with `Tab`. Subagents are
invoked by the builder explicitly when needed.

> **Why the distinct names?** OpenCode merges agent definitions from
> all `.opencode/agents/` directories along the path (root + service)
> into a single namespace. If both root and service had agents named
> `planner`, only one would be loaded — collision. The `-root` suffix
> keeps them distinct so each level loads its intended agents.

### Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  Session start → planner (default agent)                         │
│  │                                                               │
│  ├─ Configuration sync: resolve any unsubstituted placeholders   │
│  ├─ Triage: assess state, detect next action                     │
│  │                                                               │
│  ├─ Setup mode      → architect-service skill → specs + arch    │
│  ├─ Brownfield mode → adopt SDD on existing code                │
│  ├─ Architecture    → architect-service skill → Plan section    │
│  ├─ Planning mode   → task-decomposition skill → Micro-tasks    │
│  ├─ Detailing mode  → plan next micro-task in chat              │
│  ├─ Wrap-up mode    → close macro-task: aggregate review,       │
│  │                    full test suite, consolidate module docs  │
│  ├─ Refactor mode   → user-triggered: scope change, impact,     │
│  │                    create refactor task                       │
│  ├─ Express mode    → planner-gated trivial fix (typo, bump)    │
│  │                                                               │
│  └─ Handoff to builder (Tab)                                    │
│                                                                  │
│  builder                                                         │
│  │                                                               │
│  ├─ Implement micro-task + tests (module-scoped)                │
│  ├─ Write completion log + doc-sync declaration                  │
│  ├─ Conditional: @security-reviewer if security-relevant        │
│  ├─ On-demand: @code-reviewer when user asks                    │
│  ├─ Free mode: direct-to-builder, zero ceremony, untracked      │
│  └─ Session ends. User opens fresh session for next micro-task   │
└─────────────────────────────────────────────────────────────────┘
```

### Skills

Skills encode **methodology**, loaded on-demand by agents:

| Skill                         | Level   | When used                                     |
| ----------------------------- | ------- | --------------------------------------------- |
| `analyst`                     | root    | Eliciting project requirements (PDR)          |
| `architect-project`           | root    | Designing cross-service architecture          |
| `architect-service`           | service | Designing internal architecture (3 modes)     |
| `task-decomposition`          | service | Breaking macro-tasks into atomic micro-tasks  |
| `<stack-name>` (user-provided)| service | Stack-specific conventions (optional)         |

### Documentation structure

The `.docs/` directory holds project knowledge at two levels:

**Project level (`.docs/` at root)** — auto-loaded for project sessions:
- `PDR.md` — requirements
- `architecture.md` — cross-service design
- `infra.md` — environments, deployment
- `contracts/*` — inter-service interface definitions

**Service level (`<service-dir>/.docs/`)** — auto-loaded for service
sessions, partitioned per agent (planner sees more, builder/reviewers
see less):
- `specs.md` — what the service does
- `architecture/architecture.md` + `modules/<n>.md` — internal design
- `infra/server.md` + `infra/ops.md` — runtime and deployment
- `security-issues.md` — service security registry
- `tasks/tasks.md` — macro-tasks index
- `tasks/task-NN-<n>.md` — single ledger per macro-task with Plan,
  Micro-tasks, Emergent decisions, Execution logs, Wrap-up sections

Files frequently updated (task ledgers, decisions) are lazy-loaded —
the agent reads them when navigation requires.

### Operating modes

The planner's behavior is shaped by what it detects at triage. Each mode
has a clear purpose:

| Mode           | Trigger                                                | Output                                          |
| -------------- | ------------------------------------------------------ | ----------------------------------------------- |
| Setup          | Foundational docs missing                              | `specs.md` + `architecture.md`                  |
| Brownfield     | User says "modalità adozione" / "brownfield"           | Baseline docs from existing code                |
| Architecture   | New macro-task, no Plan section yet                    | Plan section of macro-task ledger               |
| Planning       | Plan approved, no Micro-tasks section yet              | Micro-tasks section                             |
| Detailing      | Micro-task ready to start                              | Implementation plan in chat (not on disk)       |
| Wrap-up        | All micro-tasks of a macro-task done                   | Aggregate review, full tests, module doc update |
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
- **Two-tier test policy** — module-scoped tests during each
  micro-task, **mandatory full test suite** at wrap-up. Wrap-up is
  blocked if the suite fails.
- **Code review on-demand, security review automatic** — code-reviewer
  is invoked only when the user asks (or at wrap-up for aggregate
  review). Security-reviewer fires automatically when a micro-task is
  flagged `security-relevant: true`.
- **Documentation sync mandatory** — builder declares updates,
  code-reviewer verifies (when invoked), planner cross-checks at
  wrap-up.
- **Cross-service changes stop the service-level session** — when a
  change requires touching an inter-service contract, it must be
  handled from project root.
- **Brownfield-friendly** — adoption on existing codebase doesn't
  require rewriting docs. The planner has a dedicated brownfield mode
  that produces a baseline; module docs accumulate lazily.
- **Significant changes have a dedicated flow** — refactor mode in 3
  phases (scoping, impact mapping, create refactor task). The refactor
  task's Plan section is the source of truth for target architecture.
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
│   ├── root/
│   │   ├── AGENTS.md
│   │   ├── opencode.json
│   │   └── .opencode/
│   │       ├── agents/
│   │       │   ├── planner-root.md
│   │       │   └── builder-root.md
│   │       └── skills/
│   │           ├── analyst/SKILL.md
│   │           └── architect-project/SKILL.md
│   └── service/
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
│               └── task-decomposition/SKILL.md
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
