---
name: wrap-up
description: Step-by-step procedure of the planner's macro-task wrap-up (aggregate reviews, full test suite, CI gate, architecture consolidation, emergent decisions, doc-sync, compaction, wrap-up commit, integration). Load it whenever a macro-task is wrap-up-pending and execute every step in order.
---

# Macro-task wrap-up (planner)

You are the `planner` and a macro-task is `wrap-up-pending`. The exit
preconditions are in your agent prompt (**Wrap-up mode**): you may not
mark the macro-task `done` until each one is verified by an artifact.
Execute the steps below **in order**; never skip one silently.

## Steps


 1. **Aggregate review**: read the macro-task ledger (Plan, Micro-tasks, Emergent decisions) and its execution log (`<id>.log.md`).
    Summarize in chat the overall scope of what
    was built and how it compares to the original plan. Note any
    divergences from plan (features added/removed/modified during
    implementation).
2. **Aggregate code review (mandatory)**: invoke `@code-reviewer` via
   the `subagent` tool with explicit instruction to review the **aggregate
   diff of the entire macro-task**, not a single micro-task. This is
   the macro-level quality gate that compensates for the per-micro-task
   review being on-demand during implementation. The code-reviewer will:
   - Inspect the cumulative diff of all micro-tasks of this macro
   - Check for cross-cutting issues that single-micro-task reviews
     would have missed (orphaned helpers, public methods no longer
     called, configuration keys defined but not read, accumulated
     dead code, naming inconsistencies, duplications)
   - Apply fixes directly where appropriate, or flag for user decision
   - Append an "Aggregate code review" section to the execution log
     `<id>.log.md` (under `## Wrap-up — <date>`); you copy its verdict
     line into the ledger's **Wrap-up** section
   If the code-reviewer modifies code, the user is informed and asked
   to verify (since builder is no longer running tests at this point —
   wrap-up is post-implementation). For non-trivial changes, the user
   may request a manual test run before proceeding.
   This review is a **non-skippable gate**: its verdict line
   must exist in the ledger's Wrap-up section before you can transition the macro-task to
   `done`. If for any reason you cannot invoke `@code-reviewer`, **hard-stop
   and ask** the user to run it (manually, or via a `builder` session) —
   never proceed as if the review had passed. When the micro-tasks touched
   `security-relevant` areas, invoke `@security-reviewer` on the aggregate
   diff under the same gate.
3. **Full test suite verification (mandatory)**: during implementation
   each micro-task ran a **subset** of tests (typically tests of the
   affected module). Wrap-up is the moment to verify there are no
   cross-functional regressions: run the full test suite of the
   service. If the suite passes, proceed. If it fails:
   - **Block the wrap-up**. The macro-task stays at `wrap-up-pending`.
   - Surface the failures clearly. Distinguish between failures in
     modules touched by this macro-task (a regression introduced
     here) and failures in unrelated modules (potentially a
     regression that escaped earlier wrap-ups, or a new flaky test).
   - Propose a path forward to the user: spawn a follow-up
     micro-task to fix the regression, or mark a previously `done`
     micro-task as `blocked` if the failure originates from there
     and needs revision.
    - Do not proceed until the suite passes (or the user explicitly accepts
      the failures with documented reason). A wrap-up that "completes" while
      reporting red tests is a defect — **fix, don't defer**.
3b. **CI gate (only when CI is enabled)**: read `ci.provider` in
   `../sdd.config.yaml`.
   - `none`, or the file is missing → the CI gate **does not exist**.
     State "CI not enabled — CI gate skipped" and proceed. The full test
     suite (step 3) still applies.
   - Otherwise, reproduce locally the checks the CI pipeline enforces,
     so a macro-task is never closed in a state the pipeline would
     reject. Read the pipeline definition (`.github/workflows/*.yml`,
     `.gitlab-ci.yml`, or `ci.files`) and **derive the exact checks**
     for this service: lint, type-check, test, build, custom steps. The
     pipeline definition is the single source of truth — do not invent
     a different set. Run them in pipeline order. Any failure **blocks
     the wrap-up** exactly as in step 3 (fix, don't defer).
 4. **Consolidate module architecture**: this is the key architectural
   step of wrap-up. **Invoke `architect-service` skill in "macro-task
   consolidation" mode**. The skill guides you through:
   - Reading the Plan section + actual implemented code
   - Identifying which architectural decisions from the Plan are now
     stable facts vs which evolved during implementation
   - Creating or updating `.docs/architecture/modules/<name>.md` for
     any new modules introduced
   - Updating existing `modules/*.md` for modules that were changed
   - Updating `.docs/architecture/architecture.md` if there are
     service-level implications
   The stable state of the module(s) is what gets documented here —
   the Plan section remains as historical intent.
 5. **Review emergent decisions**: if the Emergent decisions section is non-empty,
    review its entries. These are the emergent decisions and
    discoveries that surfaced during detailing and implementation
    (things the Plan did not anticipate). For each entry, decide:
    - If it's purely tactical and already absorbed by the final code,
      leave it in the ledger's Emergent decisions section as
      historical record — no further action
   - If it has implications that should outlive this macro-task,
     propose promoting it to `.docs/architecture/architecture.md` or
     to the appropriate `modules/<n>.md`
   - If it contradicts the Plan in a way that should have triggered
     a plan revision, flag this to the user as a lesson for future
     planning — but don't rewrite the Plan retroactively
6. **Doc-sync cross-check**: extract every "Documentation updates"
   entry from the micro-task logs. For each, verify the corresponding
   file actually contains the declared change (spot-check a few files
   with `grep` or `cat`). Report any discrepancy.
7. **Security issues check**: read `.docs/security-issues.md`. List any
   new issues opened during this macro-task that remain `open`. Flag
   them to the user for follow-up planning.
8. **Architecture coherence final check**: after the consolidation in
   step 4, do a final pass comparing architecture docs with the code.
   This is a verification — most drift should already be resolved by
   step 4, but catch anything missed.
9. **Dead code — verify, do not re-run**: the aggregate dead-code scan is
   owned by the **code-reviewer** in step 2 (orphaned helpers, public
   methods no longer called, config keys defined but unread, accumulated
   dead code). Do **not** repeat that analysis here — that would duplicate
   the reviewer's work. Your job is only to **confirm** the "Aggregate code
   review" section of `<id>.log.md` reports the **Dead code & legacy**
   field as done, and to handle two
   things the reviewer doesn't own: `SDD-KEEP` markers whose reason
   references a now-completed task (flag them for removal), and anything
   the user explicitly asked you to double-check. Each check has a single
   owner — the planner orchestrates and verifies, it does not re-execute
   the subagents' work.
10. **Refactor closure** (only if the macro-task is of
    type `ref`): verify that the target architecture described
    in the refactor task's Plan section has been realized in the code,
    and that `specs.md` / `architecture.md` / `modules/<n>.md` now
    reflect the refactored state (these were updated in step 4 via the
    architect-service consolidation mode). Then check `tasks.md` for any
   macro-tasks that were marked `blocked` during the refactor's impact
   mapping and that depended on this refactor — transition them back
   to `pending` or `in-progress` as appropriate. Finally, clean up any
   `SDD-KEEP` markers introduced during the refactor for parallel-
   implementation transitions.
11. **Compaction and closure** — run the **Wrap-up compaction** procedure
   of the `task-decomposition` skill: write the ledger **Summary**
   (≤ 30 lines) and **Wrap-up** outcome lines; set the status to `done`
   and turn the `tasks.md` entry into its one-line `Completed` form;
   append the one-line `wrapped up` entry to `task-log.md`; check that
   the living docs describe the current state only. Surface the next
   macro-task in the queue.
12. **Propose the wrap-up commit (mandatory exit precondition)**: gather
   every file modified during wrap-up (ledgers, `tasks.md`, `task-log.md`,
   architecture docs, `security-issues.md`, etc.) and **propose a commit**
   to the user — do not end the session with a dirty working tree and no
   proposal.
   - Build a Conventional Commits message scoped to the artifact
     (`docs(tasks): wrap-up <id> — <summary>`), with a footer
     referencing the ledger.
   - Show the message and the files to be staged, and wait for
     confirmation.
   - On confirmation, run `git add -- <files> && git commit -m "<message>" -- <files>`
     (explicit paths, never `-A`); report the hash.
   - If the user declines, skip the commit but leave the tree ready.
13. **Integrate (per `../sdd.config.yaml`)**: load the **`git-workflow`**
   skill and run its **End of wrap-up — integration** step: push, pull
   request, CI checks, merge on user approval for `git.remote: github`;
   push only for `other`; local merge on approval or nothing for `none`.
   - **Red CI** on the pull request blocks the integration — never merge
     red. Reopen the macro-task: add a fix micro-task (`mt-NNa`) to the
     ledger, set the status back to `in-progress` in the ledger header
     and in `tasks.md` (the line goes back to `Active`), commit this
     change, and stop. The fix runs in the normal flow; the next wrap-up
     repeats the gates and the integration.
   - **Integration deferred** by the user: keep the macro-task `done`,
     append ` — not integrated` to its `Completed` line in `tasks.md`.
     Planner triage proposes the integration in a later session.

End the wrap-up session after these steps are confirmed by the user.
