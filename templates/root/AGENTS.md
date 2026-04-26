# Project — <Project Name>

> Spec-Driven Development workspace. This file is your mental map of the
> project: where things live, how the workflow runs, what your boundaries are.
> The actual project knowledge lives in `.docs/` and is auto-loaded via
> `opencode.json` (`instructions` field) — you don't need to manually open it.

## What this project is

<One short paragraph: domain, purpose, who it's for.>

## Tech & topology

- **Architecture**: <e.g. microservices monorepo, N services + a frontend>
- **Primary stacks**: <language/runtime + framework per each service/frontend>
- **Inter-service comms**: <e.g. HTTP/REST, gRPC, async messaging — specify>
- **Deployment target**: <e.g. on-prem cluster, cloud managed service, etc.>

## Repository layout

```
.
├── AGENTS.md                      # ← you are here
├── opencode.json                  # eager-loaded project context + default_agent
├── .opencode/
│   ├── agents/
│   │   ├── planner.md             # primary agent — heavy model, doc work
│   │   └── builder.md             # primary agent — light model, code work
│   └── skills/
│       ├── analyst/               # methodology: requirements elicitation → .docs/PDR.md
│       └── architect-project/     # methodology: cross-service architecture
├── .docs/                         # project-level knowledge (auto-loaded)
│   ├── PDR.md
│   ├── architecture.md
│   ├── infra.md
│   └── contracts/
│       └── <contract files>       # e.g. OpenAPI, AsyncAPI, protobuf — whatever fits
└── <service-dir>/                 # individual services, each self-contained
    └── <see service AGENTS.md>
```

## Documentation map

The following files are **already in your context** (loaded via `opencode.json`).
Do not re-read them unless you need to verify a specific section.

| File                          | What it contains                                                |
| ----------------------------- | --------------------------------------------------------------- |
| `.docs/PDR.md`                | Functional & non-functional requirements                        |
| `.docs/architecture.md`       | Inter-service architecture, boundaries, communication patterns  |
| `.docs/infra.md`              | Server, cloud, environments, CI/CD targets                      |
| `.docs/contracts/*`           | Service contracts (source of truth for inter-service interfaces)|

## Agents at this level

- **`planner`** (default at session start) — produces and maintains
  project-level documentation. Uses the heavy/reasoning model. Loads
  skills `analyst` and `architect-project` as needed.
- **`builder`** — present but rarely used at this level. Use only for
  small direct edits to project docs that don't require methodology.
  For substantive design work, stay in `planner`.

Switch agents with `Tab`.

## Session behavior

At session start, the `planner` agent is active by default. Its default
behavior is to **perform a triage** of the project state and propose the
next action — without requiring any specific keyword from the user. A
greeting, an empty prompt, or a generic "go / let's start" is enough.

If the user's first message contains specific instructions (e.g. "review
architecture.md", "I want to redesign the auth boundary"), the planner
follows those instructions instead of running the default triage.

## Workflow at this level

You operate at this level for **project-wide concerns**: gathering
requirements, defining cross-service architecture, designing service
contracts.

The default flow on a fresh session, in `planner`:

1. **Triage**: check what project-level documentation exists and what's
   missing or outdated.
2. **Setup mode** (if something is missing): invoke `analyst` for the PDR,
   `architect-project` for architecture/infra/contracts.
3. **Stop and hand off**: once project-level docs are in shape, the actual
   implementation work happens inside individual services —
   end the session and instruct the user to `cd <service-dir> && opencode`.

> The workflow above is the **default**. User instructions during the
> conversation always take precedence — if the user wants to deviate
> (revise architecture, jump to a specific document, etc.), follow their
> lead without lecturing them about the flow.

## Boundaries

- **Do not** create or modify code inside service directories from this level.
  Service-level work happens by launching `opencode` from inside the
  service directory.
- **Do not** invent new services without first updating
  `.docs/architecture.md` and the relevant contract(s).
- **Do** ask clarifying questions when requirements (PDR) are ambiguous.
  This project values explicit decisions over assumptions.

## Session discipline

A session at this level is bounded: produce/update one coherent piece of
project documentation, then stop. Don't chain unrelated work in the same
session — open a new one.
