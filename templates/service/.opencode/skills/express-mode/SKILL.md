---
name: express-mode
description: "Express mode of the SDD pattern: the gate (when a trivial change qualifies), the planner side (one-line plan, handoff) and the builder side (implement, quality gate, express-log line, short commit). Load it when the user asks for an express/quick fix or a change is classified as trivial."
---

# Express mode

## Overview

Not every change deserves the full pipeline. For trivial work — typo
fixes, dependency bumps, config tweaks, minor renames — both planner
and builder support **express mode**, which bypasses micro-task
decomposition and documentation tracking while keeping build + tests
green and a commit proposal. Code review stays on demand.

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
- No automatic code review — the user can request it ("run code
  review" / `@code-reviewer`); if the code-reviewer modifies code,
  builder re-runs build + tests
- Builder appends a one-line entry to `.docs/express-log.md`
- Builder proposes a commit (Conventional Commits format, no SDD
  footer, just `Ref: .docs/express-log.md`) and waits for user
  confirmation — same as the normal flow
- Session ends

### What express mode does NOT produce

- No entry in any Micro-tasks section
- No entry in any execution log
- No Emergent decisions entry
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

## Planner side

For trivial changes that don't deserve the full SDD pipeline — typo
fixes, dependency bumps, config tweaks, minor renames — you enter
express mode. Express mode bypasses planning, micro-task decomposition,
and documentation tracking. Build + tests and the quality gate stay
mandatory; code review stays on demand.

### When express mode applies

Express mode is appropriate when **all** of these are true:

- The change is small and self-contained (a few lines, one or two files)
- The change does not affect specs, architecture, or module docs
- The change is not security-relevant (no auth, crypto, input validation,
  secrets)
- The change does not touch cross-service contracts
- The change does not introduce or remove a module or major flow
- The change does not modify persistence schema or require migration

If any of these fails, do **not** use express mode. Recommend the
normal flow to the user.

### Express mode flow for the planner

1. Confirm with the user that the change qualifies for express mode
   (briefly list what express mode skips: micro-task tracking, security
   review, doc updates)
2. Produce a **very brief plan in chat** (one or two sentences): what
   will change, which files are likely touched, what the expected
   outcome is
3. Hand off to the builder: "Express plan ready. Switch to the `builder`
   agent with Tab — it will implement, log the fix to
   `.docs/express-log.md`, and propose a commit."
4. End your turn

Do not touch the macro-task ledger in express mode. Express work doesn't go through the ledger at all —
no Emergent decisions entries, no updates to `tasks.md`. Express work is
intentionally lightweight.

### When the user asks for express but it doesn't qualify

If the user says "quick fix" but the request actually touches specs
or security, explain clearly why it doesn't qualify:

> This change isn't eligible for express mode because it <touches
> security / modifies specs / affects contracts>. I recommend going
> through the normal flow: we should plan it as a micro-task so the
> right review and documentation steps run. Do you want to proceed
> that way?

Do not force express mode on requests that don't fit.

## Builder side

For trivial changes that don't deserve the full SDD pipeline — typo
fixes, dependency bumps, config tweaks, minor renames — you operate in
**express mode** when the planner has entered express mode and handed
off to you, or when the user has explicitly asked for an express fix.

### Constraints — never bypass these

Express mode is **not allowed** when any of these apply:

- The change affects specs, architecture, or module documentation
- The change touches `../_project/.docs/contracts/*` or any cross-service
  interface
- The change is security-relevant (auth, authz, crypto, input
  validation, secrets handling)
- The change introduces, removes, or renames a module or major flow
- The change modifies persistence schema or introduces a migration
- The change could break backward compatibility of a public interface

If you discover mid-implementation that the change falls into any of
these categories, **stop** and tell the user:

> This change exceeds express mode scope because <reason>. I recommend
> switching to the normal flow: open a new session with the planner to
> decompose this properly.

Do not continue in express mode on a change that should have been a
normal micro-task.

### Express flow

1. Mark no status anywhere (no entry in any macro-task ledger, no `tasks.md` update — express
   work is not tracked as a micro-task)
2. Implement the change
3. Run build + tests and the quality gate (same as step 6b of the
   implementation loop). Iterate until they pass.
4. **No automatic code review**. Express mode is for trivial work where
   passing tests are the primary gate. The user can request a code
   review explicitly at any point with "run code review" — in that
   case, invoke `@code-reviewer`, re-run tests if it modifies code,
   and proceed only if everything still passes.
5. No security-reviewer in express mode (by constraint, express
   excludes security-relevant changes)
6. Append a **one-line entry** to `.docs/express-log.md` with date,
   brief description, and files touched. Create the file if it doesn't
   exist. Format:

   ```
   ## YYYY-MM-DD
   - <brief description> — <file1>, <file2>
   ```

7. **Propose the commit** to the user, on the current branch (if that is
   a protected integration branch, follow **Express changes** in the
   `git-workflow` skill). In express mode the commit message is shorter: no body, no SDD task footer. Use type + scope +
   summary, with a reference to the express log:

   ```
   chore(<scope>): <brief description>

   Ref: .docs/express-log.md
   ```

   Or use `fix`, `docs`, `build`, `refactor`, `perf` as appropriate
   for the type. The `Ref:` footer tells future readers where to
   look for context.

   Show the proposed message and the list of files to stage
   (including the updated `express-log.md`). Wait for the user's
   confirmation.

   - If confirmed: `git add -- <files> && git commit -m "<message>" -- <files>` (explicit
     paths)
   - If declined or adjusted: follow the user's instruction, same as
     in the normal flow

8. Final summary and stop:
   - Express fix: <brief description>
   - Files touched: <list>
   - Code review: on-demand (<result if user requested it, "not invoked" otherwise>)
   - Logged in `.docs/express-log.md`
   - Commit: <hash or "skipped">
   - Session ends here.

### What express mode does NOT do

- No entry in any execution log
- No entry in any Micro-tasks section
- No status transition in `tasks.md`
- No security review
- No doc-sync trigger checks (by constraint, express excludes
  doc-impacting changes)
- No Emergent decisions entry

The `express-log.md` is the only record, intentionally minimal.
