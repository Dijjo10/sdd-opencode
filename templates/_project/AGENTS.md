# Project Workspace — <Project Name>

> Spec-Driven Development workspace at **project level**. This is the
> orchestration layer of the monorepo: requirements, cross-service
> architecture, contracts, infrastructure, and project-wide scaffolding.
> Service-internal work happens in separate workspaces, one per service
> directory.

## Why this is a separate workspace

OpenCode loads a single `opencode.json` per session — the one in the
directory where you launch `opencode`, or the closest one walking up
the tree. It does **not** merge nested configs. To keep project-level
and service-level concerns properly isolated, this monorepo uses
**two kinds of OpenCode workspaces**:

- **This workspace** (`_project/`) — for project-wide concerns. Launch
  with `cd _project && opencode`.
- **Service workspaces** (one per service folder, e.g. `rag/`,
  `auth/`, `api-gateway/`) — for service-internal work. Launch with
  `cd <service-dir> && opencode`.

Each workspace has its own `opencode.json`, its own agents, its own
docs context. The two layers communicate via **artifacts on disk** —
the cross-service `architecture.md` and `contracts/*` produced by
this workspace are read by service workspaces; the service docs and
implementation are independent.

## What this project is

<One short paragraph: domain, purpose, who it's for.>

## Tech & topology

- **Architecture**: <e.g. microservices monorepo, N services + a frontend>
- **Primary stacks**: <language/runtime + framework per each service/frontend>
- **Inter-service comms**: <e.g. HTTP/REST, gRPC, async messaging — specify>
- **Deployment target**: <e.g. on-prem cluster, cloud managed service, etc.>

## Repository layout (project-level perspective)

```
<project-root>/
├── _project/                      # ← THIS WORKSPACE (project-level)
│   ├── AGENTS.md                  # ← you are here
│   ├── opencode.json              # project workspace config
│   ├── .opencode/
│   │   ├── agents/
│   │   │   └── principal.md       # the single project-level agent
│   │   └── skills/
│   │       ├── analyst/           # methodology: requirements → PDR.md
│   │       └── architect-project/ # methodology: cross-service architecture
│   └── .docs/                     # project-level knowledge (lazy-loaded)
│       ├── PDR.md
│       ├── architecture.md
│       ├── infra.md
│       └── contracts/
│           └── <contract files>   # e.g. OpenAPI, AsyncAPI
│
├── <service-dir-1>/               # service workspace (independent)
│   ├── AGENTS.md
│   ├── opencode.json
│   ├── .opencode/
│   ├── .docs/
│   └── <source-tree>/
│
├── <service-dir-2>/               # another service workspace
│   └── ...
│
├── README.md                      # root README (edited from this workspace)
├── docker-compose.yaml            # root compose (edited from this workspace)
├── .github/workflows/             # CI/CD (edited from this workspace)
└── scripts/                       # project-wide scripts
```

The `_project/` folder is **a workspace**, not a service. It owns the
**project-level docs and scaffolding**. Despite living inside a
subfolder, the principal agent in this workspace can still edit files
**outside** `_project/` when those files are project-wide concerns
(root README, root Dockerfile, CI configs, etc.) — see Boundaries.

## Documentation map

Your context at session start is `AGENTS.md` (this file) only — no
project doc is eager-loaded (the root `instructions` list is
intentionally absent in this workspace). Project docs are large and
grow over time; loading them eagerly would saturate every session,
including the trivial ones. Read them with your read tools when the
work requires them.

| File / Path                       | What it contains / when to read                            |
| --------------------------------- | ----------------------------------------------------------- |
| `../sdd.config.yaml`              | Pattern settings (CI, git, versioning) — **always** read at session start; you own this file |
| `.docs/PDR.md`                    | Functional & non-functional requirements — read when working on requirements |
| `.docs/architecture.md`           | Inter-service architecture — read when designing or changing cross-service structure |
| `.docs/infra.md`                  | Environments, deployment — read when working on infra       |
| `.docs/contracts/*`               | Service contracts (source of truth for inter-service interfaces) — read the relevant sections only |
| `.docs/source/`                   | Reference materials provided by the user (scope notes, regulations, spreadsheets) — **read-only** |
| `../<service-dir>/AGENTS.md`      | When you need to understand a service's role                |
| `../README.md`                    | When updating root project docs                             |
| `../.github/workflows/*`          | When working on CI/CD                                       |
| `../docker-compose.yaml`          | When working on root containerization                       |

> **Never load whole contracts.** An `external-<provider>.yaml` can be
> hundreds of KB (a full vendor OpenAPI spec). Read the index or the
> specific paths you need, never the whole file.

## Agents in this workspace

- **`principal`** (default at session start).
  Owns project-level concerns: PDR, cross-service architecture,
  contracts, infrastructure, project-wide scaffolding (CI/CD, root
  Dockerfile, root README). Loads skills `analyst` and
  `architect-project` as needed. Can also edit project-level code
  artifacts directly when scaffolding work is needed.

There is intentionally **only one agent** at this level. Project-level
work is mostly orchestration and design — heavy implementation belongs
to service workspaces with their dedicated `planner` + `builder` agent
pair.

## Session behavior

At session start, `principal` is active by default. Its default
behavior is to **perform a triage** of project state and propose the
next action — no specific keyword needed.

If the user's first message contains specific instructions ("review
architecture.md", "add a new service called billing", "set up CI for
the rag service"), `principal` follows those instructions instead of
running the default triage.

## Workflow

You operate at this workspace for **project-wide concerns**:

- Gathering and maintaining requirements (PDR)
- Designing cross-service architecture
- Defining and evolving service contracts
- Infrastructure and deployment topology
- Project-wide scaffolding (CI/CD pipelines, root Dockerfile,
  docker-compose at root, root README/LICENSE/.gitignore, scripts)
- Bootstrapping new service folders (skeleton only — implementation
  happens in the service workspace)

The default flow on a fresh session:

1. **Configuration sync**: resolve any unsubstituted placeholders
   in `.md` files
2. **Triage**: check what project-level documentation exists and
   what's missing or outdated
3. **Mode selection** (based on triage outcome):
   - **Setup mode** — foundational docs missing → `analyst` skill
     for PDR, `architect-project` for architecture
   - **Architecture mode** — new cross-service concern → invoke
     `architect-project`
   - **Contracts mode** — contract needs creation/update → edit
     `.docs/contracts/*` directly
   - **Scaffolding mode** — CI/CD, root Dockerfile, etc. → edit
     directly with discipline
   - **Bootstrap-new-service mode** — create new service folder
     skeleton, update architecture.md
4. **Hand off when appropriate** — once project-level docs are in
   shape and the user wants to implement service code, end the
   session and instruct: `cd <service-dir> && opencode`.

> The workflow above is the **default**. User instructions during
> the conversation always take precedence — if the user wants to
> deviate, follow their lead without lecturing them about the flow.

## Boundaries

You **may edit** files outside this workspace, but only if they are
**project-level**:

- `../README.md` (root README)
- `../LICENSE`, `../.gitignore`, `../.editorconfig`
- `../Dockerfile` and `../docker-compose.yaml` at root
- `../.github/workflows/*` (or `../.gitlab-ci.yml`, etc.)
- `../scripts/*` (project-wide scripts)
- New service folder skeletons (creating empty service directories
  with template files when bootstrapping)

You **must not edit**:

- Source code or tests inside any service folder
- Service-internal docs (`<service>/.docs/*`)
- Service-internal config (`<service>/opencode.json`,
  `<service>/.opencode/*`)
- Service-internal `Dockerfile` if a service has its own

If the user asks you to modify something service-internal, stop and
direct them to the service workspace:

> This request is service-internal. Please open a session inside
> `<service-dir>/` (e.g. `cd ../<service-dir> && opencode`) where
> the service-level agents have the proper context.

## Session discipline

A session at this workspace delivers **one coherent piece of
project-level work**. Examples of a bounded session:

- Produce or revise the PDR
- Add a new cross-service requirement and propagate to architecture
  and contracts
- Bootstrap a new service folder (skeleton + architecture entry +
  contract stub)
- Set up the CI/CD pipeline
- Update root README after a milestone

Don't chain unrelated work in the same session. If the user asks for
a second, unrelated deliverable, suggest opening a new session.

## Clarification principle

When requirements are ambiguous, **ask before writing**. This project
values explicit decisions over assumptions. A question in chat is
cheaper than a doc that needs to be revised.
