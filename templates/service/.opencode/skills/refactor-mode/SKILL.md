---
name: refactor-mode
description: "Refactor mode of the SDD pattern: classifying a significant change of direction (narrow/medium/wide), impact mapping, superseding decisions without losing history, and creating the `ref` macro-task. Load it when the user signals a change of direction that one micro-task cannot absorb."
---

# Refactor mode

## Overview

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
  macro-task of type `ref` (e.g. `07-ref-persistence`).
- **Wide**: service-level architecture, possibly cross-service. Must
  start with a session from the `_project/` workspace to align the cross-service
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
(type `ref` in the task ID) and an up-front impact mapping phase that
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

## Planner procedure

Activated by explicit user intent when there is a significant change of
direction that can't be absorbed by a single micro-task: specifications
changing, architectural patterns being replaced, decisions being
invalidated, broad code reshaping.

A refactor is **just a macro-task with corrective intent** — same
lifecycle (architecture mode → planning → implementation → wrap-up),
with two specializations:

1. Task type `ref` in the ID (`NN-ref-<slug>`) so refactors are
   recognizable at a glance
2. Up-front impact mapping to mark superseded decisions and abandoned
   work without losing history

Refactor mode runs in 3 phases; after phase 3 the regular workflow takes
over (next session enters architecture mode for the new refactor task).

**Phase 1 — Scoping.** Classify the refactor with focused questions:
*Narrow* (one spec/flow, fits one macro-task) → **don't enter refactor
mode**, handle as normal re-planning. *Medium* (architecture of one+ modules
in this service, invalidates past decisions) → refactor mode applies,
proceed. *Wide* (service-level architecture: layering, persistence,
comms — may impact other services) → **pause**: tell the user to align
architecture and contracts from the `_project/` workspace first, then return.

**Phase 2 — Impact mapping.** Produce an **impact report in chat** (not on
disk) listing what the refactor invalidates: affected decisions (in Plan /
Emergent decisions of active or past ledgers), affected macro-tasks
(`in-progress`/`pending` now invalidated or needing revision), affected
micro-tasks (`done` ones now obsolete, `pending` ones needing revision).
After user confirmation, apply state changes: in `tasks.md` mark affected
macro-tasks `blocked` (will resume) or `skipped` (abandoned) with a one-line
reason; mark affected micro-tasks likewise in their ledgers; for each
superseded decision set `Status: superseded by NN-ref-<slug>` plus a
one-line "Superseded because" note. **Never delete original decision text** —
it stays as history. Do **not** touch `specs.md`/`architecture.md`/`modules/*`
now: they keep describing current code until the refactor's wrap-up updates
them. Until then the refactor task's Plan section is the source of truth for
the target.

**Phase 3 — Create the refactor macro-task.** Add it to `tasks.md`: ID
`NN-ref-<slug>` (the `ref` type is mandatory),
status `pending`, one-line objective (the target, not the abandoned state),
and `Depends on` any blocked macro-tasks that resume afterward. Don't write
the Plan yet — that's `architect-service`'s job next session. End the session
telling the user to open a new one for architecture mode.

The refactor task then follows the standard flow; its **wrap-up** updates
`specs.md`/`architecture.md`/module docs to the refactored state, resumes
`blocked` dependent macro-tasks, and cleans up `SDD-KEEP` markers introduced
for parallel-implementation transitions.
