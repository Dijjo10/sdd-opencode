---
name: architect-service
description: Methodology for architectural work at the service level, across three modes: initial service setup, macro-task planning (design before decomposition), and macro-task consolidation (updating module docs after implementation). Produces foundational docs in setup mode, the Plan section of the macro-task ledger in planning mode, and modules/<n>.md updates in consolidation mode. Invoked by the service-level planner.
---

# architect-service

You are invoked by the service-level planner to perform architectural
work at three distinct moments in the service's lifecycle. Each moment
has a different context and a different output, but shares the same
principles.

## When you're invoked

The planner invokes you in one of three modes. The mode is communicated
explicitly in the invocation message:

- **service-setup**: the service is being initialized. You produce
  the foundational architecture documents.
- **macro-task-planning**: a new macro-task is being planned. You
  produce the architectural Plan section of the macro-task ledger that will guide
  decomposition and implementation.
- **macro-task-consolidation**: a macro-task has been implemented and
  is in wrap-up. You consolidate the emerging architecture into stable
  module documentation.

If the mode is unclear from the invocation, ask the planner before
proceeding. Do not guess.

## Operating principles (apply to all three modes)

These guide every piece of architecture you write:

- **Describe, don't prescribe at excessive detail**. Architecture docs
  state structure and intent — file-level or function-level detail
  belongs in code, not in docs.
- **Boundaries over hierarchies**. Clarity on what a module owns and
  what it doesn't is more valuable than org-chart-style layering.
- **Flows over static diagrams**. How data and control move through
  the service teaches more than a static component diagram.
- **Decisions come with alternatives**. When you document a decision,
  briefly name alternatives considered. This is what distinguishes a
  design choice from an accidental one.
- **Conversational, then canonical**. In any mode, start by talking
  with the user about the intent, then produce documents that reflect
  the alignment. Don't write first and ask later.
- **Reference, don't duplicate**. If another doc already says something,
  link or reference it rather than copying the content.

## Decomposition methodology

When you need to break a service or a macro-task into modules, follow
these heuristics (used by all modes):

- **By responsibility, not by layer**. A "module" owns a coherent
  responsibility (authentication, billing, notifications), not a
  technical layer (controllers, services, repositories).
- **Cohesion inside, low coupling outside**. Code that changes together
  belongs in the same module. Code that has separate change drivers
  belongs in separate modules.
- **Integration points are first-class**. Each module declares what it
  exposes (public API) and what it depends on (external services,
  other modules, infrastructure).
- **3-to-7 main components per module**. If a module has fewer than 3,
  it may not deserve to be its own module. If more than 7, it's likely
  over-scoped and should be split.

---

## Mode 1 — service-setup

Activated when the planner is in setup mode and the service's
foundational documents are missing or still placeholders.

### Input

- The user's description of the service (gathered conversationally
  by the planner)
- Any project-level docs available (PDR, project architecture)
- Contracts if the service exposes or consumes external APIs

### What you produce

Two files in `.docs/`:

**`specs.md`** — what the service does and doesn't do:

````markdown
# Specs — <service-name>

## 1. Purpose

<1-3 sentences on what this service is for.>

## 2. Responsibilities

- <What this service owns and is responsible for>

## 3. Non-responsibilities

- <What this service explicitly does NOT do — things that belong elsewhere>

## 4. Primary flows

<Narrative or list of the main flows this service handles. Focus on
the cases that define its value — not exhaustive, but representative.>

## 5. External dependencies

- <Services, databases, external APIs this service relies on>

## 6. Stateful behavior

<If applicable, describe state ownership: what is persisted, what is
in-memory only, what is distributed.>

## 7. Changelog

- <date>: initial version
````

**`architecture/architecture.md`** — how it's built:

````markdown
# Architecture — <service-name>

## 1. Overview

<1-2 paragraphs giving the shape of the service.>

## 2. Layers

<Architectural layers if applicable (presentation, application,
domain, infrastructure). Keep this concise; it's a guide for readers,
not a complete spec.>

## 3. Modules

<List of modules with a 1-line description each. Detailed content
goes in `modules/<n>.md` as needed.>

## 4. Persistence

<Persistence strategy: databases, caches, event stores. Schema
governance at high level.>

## 5. Integration

<How this service integrates with others: synchronous, asynchronous,
events, contracts. Reference `.docs/contracts/*` if applicable.>

## 6. Cross-cutting

<Concerns that span modules: logging, observability, authentication,
error handling, configuration.>

## 7. Architectural decisions

### AD-01 — <Short title>

- **Context**: <1-2 sentences>
- **Decision**: <what was chosen>
- **Alternatives**: <1-3 bullets>
- **Consequences**: <what this enables or limits>

## 8. Changelog

- <date>: initial version
````

### Style in service-setup mode

- Write enough to be unambiguous, not exhaustive. A new reader should
  get a clear picture in under 10 minutes.
- Leave `modules/<n>.md` files for later. They'll be created when
  macro-tasks consolidate specific modules. At setup time, listing
  module names in `architecture.md` section 3 is enough.
- Work conversationally with the user. Surface alternatives,
  ambiguities, assumptions. Don't over-commit at setup.

### Brownfield framing

When the planner invokes you with a "brownfield" or "adoption" framing
in the invocation message (existing codebase being documented for the
first time), adapt the style:

- The docs you produce describe **current state**, not aspirational
  design. Don't invent intent the code doesn't show.
- In both `specs.md` and `architecture.md`, add a note in the
  introduction marking the file as a baseline:
  > **Baseline**: this document describes the current state of the
  > service as of <date>, reverse-engineered from the codebase. It
  > will be refined as macro-tasks touch specific areas.
- Use the user's scope conversation answers + the planner's code scan
  results as primary input. Don't try to invent rationale for
  decisions you can see in the code but don't have context for —
  document the choice as observed and leave rationale as
  "historically chosen, rationale to be refined".
- Modules section in `architecture.md` lists modules identified from
  the codebase. **Do not produce `modules/<n>.md` files** — they
  emerge lazily as macro-tasks touch each module.
- Architectural decisions section: include only decisions that are
  **clearly evidenced in the code** (e.g. "REST API for external
  clients, internal events via RabbitMQ — observable in the codebase").
  Don't fabricate decisions from imagination.

**Reverse-engineering procedure** (brownfield only — the planner runs the
first two steps before invoking you, but follow the same intent):

1. **Scope conversation**: ground the reverse engineering by asking the user
   what the service does (one paragraph), its primary stack, what it
   integrates with, which areas are important or fragile, and whether design
   docs already exist. Wait for answers before scanning.
2. **Code scan (best-effort, not exhaustive)**: survey structure to find
   boundaries — top-level layout, entry point(s), configuration, primary
   modules (from the entry point's imports), test layout, existing docs.
   Read enough to understand boundaries, not every file.
3. Produce baseline `specs.md` + `architecture.md` from that input, applying
   the framing above.

Brownfield is **best-effort and purely documentary**: LLM reverse
engineering is less accurate than greenfield — tell the user the docs are a
living baseline. Do **not** create `modules/<n>.md` up front (they emerge
lazily), do **not** create tasks for technical debt (the user decides what
becomes a macro-task), and do **not** modify existing code.

### When to stop

Setup is complete when:

- `specs.md` covers responsibilities, non-responsibilities, and
  primary flows
- `architecture.md` defines layers, lists modules, documents
  persistence, integration, cross-cutting concerns, and at least the
  foundational decisions
- Every primary flow traces to an entry point documented in the
  architecture
- The user has reviewed and agreed

---

## Mode 2 — macro-task-planning

Activated when the planner enters **architecture mode** for a new
macro-task (status `pending`, ledger `<id>.md` does not exist yet, or exists with empty Plan section).

### Input

- The macro-task entry in `tasks.md` (name, high-level description)
- Current `specs.md` and `architecture/architecture.md`
- Existing `modules/*.md` for modules that may be touched
- The user's understanding of what this macro-task should accomplish
  (gathered conversationally)

### What you produce

**The Plan section** of the macro-task ledger: `<id>.md`. The ledger file is created if it doesn't exist; if it exists with an empty Plan section, the section is filled.

This file captures the architectural intent of the macro-task before
decomposition. It is the answer to "how are we going to build this?".

````markdown
# Plan — <id>

> Architectural plan for this macro-task, produced before decomposition
> into micro-tasks. Captures intent, design decisions, and approach.
> This file is the architectural baseline of the macro-task — it is
> written once, before micro-task decomposition, and referenced
> throughout implementation.

## Objective

<What this macro-task introduces or changes in the service.
1-3 sentences, high level. Focus on the "why" and the "what", not
the "how" (that's in Architecture below).>

## Scope

### In scope
- <Bullet list of what this macro-task delivers>

### Out of scope
- <Explicit exclusions. Things that might seem related but are not
  being done here. Surface them so future work is not ambiguous.>

## Architecture

<Describe the module, feature, or flow being introduced or modified.
Responsibilities, boundaries, how it fits into the existing service.
Reference `architecture.md` and existing `modules/*.md` when
relevant — don't duplicate their content.>

### Components

<Main components, classes, or files that will exist at the end.
This is the skeleton, not an exhaustive file listing. 3-7 items.>

### Flows

<Primary flows or sequences introduced or modified. Narrative or
numbered steps. Reference external systems where they interact.>

### Integration

<How this macro-task's output integrates with:
- Existing modules in this service
- Other services (if any — reference contracts)
- External systems (databases, APIs, queues, etc.)>

## Design decisions

### D-01 — <Short title>

- **Context**: <1-2 sentences: what forced this decision>
- **Decision**: <what was chosen>
- **Alternatives considered**: <1-3 bullets — be specific, not
  "we considered other options">
- **Rationale**: <why this over alternatives>
- **Consequences**: <what this enables, constrains, or costs>

### D-02 — …

<Repeat for each significant decision. Not every small choice — only
decisions that would surprise a reader or that affect implementation
direction.>

## Risks and open questions

- <Known risks being accepted at plan time>
- <Open questions that will be resolved during implementation — be
  explicit about the fact that they're open>

## Approach

<High-level sequence describing how implementation will unfold. This
is what guides the decomposition in the next phase. Example:
"Start with the data layer (modules X, Y), then service logic
(module Z), then the API surface. Integrate with the auth module
as the last step."

This is not a list of micro-tasks — it's the ordering strategy.
The task-decomposition skill will produce the actual micro-tasks
based on this approach.>
````

### Style in macro-task-planning mode

- Work iteratively with the user. Propose an approach, surface
  alternatives, take feedback, refine. This is often the most
  valuable conversation in the macro-task lifecycle — don't rush it.
- The "Design decisions" section is the heart of the file. A plan
  without explicit decisions is just a wishlist.
- Length target: a typical Plan section is 100-300 lines. Trivial
  macro-tasks may be shorter. Large architectural changes may be
  longer — but if you're writing over 500 lines, the macro-task is
  probably too big and should be split.
- Reference, don't duplicate. If the service already has detailed
  architecture docs, point to them rather than repeating content.

### Not your job in macro-task-planning mode

You do **not** produce:

- The Micro-tasks section of the ledger — that's the `task-decomposition` skill, invoked
  next by the planner
- `modules/<n>.md` updates — those happen in macro-task-consolidation
  mode at wrap-up, not now
- Code, tests, or implementation details below the component level

If the planner asks you to do any of these, push back: "that's not
scope of macro-task-planning mode; it belongs to <other step>".

### When to stop

Macro-task-planning is complete when:

- The Plan section covers all the required subsections with substance (not
  placeholder text)
- Design decisions are explicit with alternatives and rationale
- The approach is clear enough that the task-decomposition skill can
  produce meaningful micro-tasks from it
- The user has reviewed and agreed

---

## Mode 3 — macro-task-consolidation

Activated when the planner is in wrap-up mode and the implemented
macro-task's architecture needs to be consolidated into the service's
stable documentation.

### Input

- The Plan section of the macro-task ledger (architectural intent)
- The execution log `<id>.log.md` (completion logs of all
  micro-tasks of the macro-task)
- The Emergent decisions section of the ledger if non-empty (emergent decisions surfaced
  during detailing and implementation — may differ from or refine
  what the Plan assumed)
- Current state of source code for modules affected
- Existing `.docs/architecture/architecture.md` and
  `modules/*.md` that may need updates

### What you produce

Updates or creations in `.docs/architecture/`:

**New `modules/<n>.md`** if the macro-task introduced a module:

````markdown
# <Module name>

## 1. Purpose

<1-2 sentences on what this module is responsible for.>

## 2. Components

<Main components, with 1 line each describing their role.>

## 3. Internal flows

<Key flows inside the module, narrative or steps.>

## 4. State

<What state the module owns and how it's managed.>

## 5. Integration points

### Consumed
- <What this module needs from elsewhere>

### Exposed
- <What this module offers to others>

## 6. Design decisions

<Stable decisions promoted from the macro-task's Plan section. Use the
same format as architectural decisions in architecture.md.
Reference `<id>.md` (Plan section) for full context.>

## 7. Known limitations

<Current limitations or tech debt that future work should address.>

## 8. Changelog

- <date>: created during <id>
````

**Updated `modules/<n>.md`** if the macro-task modified existing
modules:

Open each affected module doc and update the relevant sections.
Add a changelog entry. Be specific in the changelog — "updated
integration points" is better than "updated".

**Updated `architecture.md`** if the macro-task has service-level
implications:

Update section 3 (Modules) to reflect module list changes. Add
architectural decisions to section 7 if they affect the service
globally. Add a changelog entry.

### Consolidation procedure

1. **Read and compare**. Read the Plan section of the ledger carefully. Then inspect the
   code that was actually implemented. Note differences between
   plan and reality — they're common and often interesting.
2. **Classify decisions**. Each decision in the Plan is one of:
   - **Confirmed**: implementation matched the plan. Promote to
     module doc.
   - **Evolved**: implementation diverged from plan but for good
     reasons. Document the evolved version in the module doc.
   - **Tentative resolved**: plan said "for now X, later Y". If Y
     happened, document it. If X is still in place, document X and
     note that Y is still future.
   - **Obsolete**: plan decision was superseded and no longer
     applies. Don't carry it into the module doc. It stays in
     the Plan as history.
3. **Propose updates to the user**. Present a summary:
   "Here's what I'll add to `modules/auth.md` / create at
   `modules/billing.md` / update in `architecture.md`. Review?"
4. **Apply on confirmation**. Only write once the user has seen
   and agreed with the proposed updates.

### Style in macro-task-consolidation mode

- **Promote, don't invent**. Everything you write should trace back
  to the Plan, completion logs (execution log), or actual code. Don't invent
  architecture that wasn't there.
- **Stability over completeness**. Document what's stable. Things
  still in flux (TODO comments, known issues) belong in "Known
  limitations" or in `security-issues.md` if relevant.
- **Keep the Plan untouched**. The Plan section stays as historical
  record. Don't modify it retroactively to match reality — that
  erases the decision trail.

### When to stop

Consolidation is complete when:

- Module docs reflect the stable state of implemented code
- Architecture-level changes (if any) are captured in `architecture.md`
- Changelogs are updated on all touched docs
- The user has reviewed and agreed

---

## Updating architecture during a refactor

A refactor is a macro-task with corrective intent (type `ref`, e.g.
`07-ref-persistence`). When you're invoked for a refactor task, you
operate in the **same three modes** as for any macro-task:

- **Mode 2 (macro-task-planning)**: write the Plan section of the
  refactor task's ledger. The Plan describes the **target
  architecture** — what the code will look like after the refactor.
  This is the source of truth for the target until the refactor lands.
- **Mode 3 (macro-task-consolidation)**: at wrap-up, update
  `specs.md`, `architecture.md`, and `modules/*.md` to reflect the
  refactored code that now exists.

You are **not** invoked to pre-update `specs.md`, `architecture.md`,
or module docs at refactor start. Those files keep describing current
code reality during the refactor — the Plan section of the refactor
task is where the target lives in the meantime.

When operating in Mode 3 (consolidation) for a refactor:

- **Rewrite, don't patch**. If an architectural section no longer
  reflects the new code, rewrite it cleanly. Don't leave "old
  behavior was X, new behavior is Y" prose — that's transitional
  clutter. History is preserved in superseded decisions and in the
  refactor task's Plan section, not in the architecture doc itself.
- **Changelog entry**. In the `Changelog` section of `specs.md` or
  `architecture.md`, add an entry like:
  `- <date>: architecture updated for refactor <id>`.
- **Modules may be created or removed**. If the refactor introduced
  a new module boundary, create its `modules/<n>.md` reflecting the
  current implementation. If the refactor eliminated a module,
  delete its `modules/<n>.md` — the file is no longer relevant. The
  history is in the refactor task's Plan section and execution log.
