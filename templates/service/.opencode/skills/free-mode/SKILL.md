---
name: free-mode
description: "Free mode of the SDD pattern: untracked changes made directly with the builder, without planner, ledger or review. Load it when the builder is invoked directly with an implementation request and no planner plan or express handoff."
---

# Free mode

## Overview

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

## Builder procedure

For changes the user wants to make **without any SDD ceremony**:
exploratory edits, prototypes, personal tweaks, anything where the
overhead of express mode is still too much.

Free mode is triggered when the user opens a session, switches
directly to you with `Tab` (or launches with you as default), and
gives you an implementation request — without going through the
planner. Detection is described in `builder.md`, **Session start behavior →
Scenario C**.

### When free mode applies

Free mode is the user's explicit choice to skip the SDD flow. There
are no eligibility constraints on Claude's side: if the user activates
free mode, they have decided the change doesn't require SDD discipline.

That said, if the user's request **clearly** describes something that
would benefit from the SDD flow — a non-trivial feature, an
architectural change, a security-relevant modification, a cross-service
contract change — you may **suggest** (not enforce) using the planner
instead:

> Note: this looks like it might benefit from the standard SDD flow
> (architecture planning, decomposition, review). I can still proceed
> in free mode if you confirm, or you can switch to the planner with
> `Tab`.

Wait for the user's choice. If they confirm free mode, proceed.

### Free flow

After detecting free mode and getting user confirmation:

1. **Implement the change** as requested. Use the same tools and
   discipline you'd use in normal mode (good code, sensible commits,
   running tests if relevant) — but no SDD artifacts.
2. **Run tests if relevant.** Use module-scoped selection by default
   (same logic as Test execution policy section). If the user
   explicitly says "no tests" or the change is trivial, you can skip.
3. **No completion log.** No subsection added to any ledger.
4. **No review chain invocation.** Neither code-reviewer nor
   security-reviewer fires automatically.
5. **No express-log entry.** Free mode is intentionally untracked.
6. **Commit, if appropriate.** Either propose a commit (Conventional
   Commits format, no SDD task footer, no `Ref:` line) or leave the
   working directory dirty for the user to commit themselves —
   ask their preference if not specified.

### What free mode does NOT do

- No status transitions anywhere
- No documentation sync
- No reviewer invocations
- No tracking artifacts (not even an express-log line)
- No coupling to the macro-task ledger system

### Reverting to standard flow

If during free mode work the user changes their mind ("actually let's
plan this properly"), they can switch back to the planner with `Tab`.
You stop, summarize what you've done so far in chat (so the planner
has context), and the user proceeds with the standard SDD flow.

If you've already made code changes in free mode that the user now
wants to continue under SDD, the planner will absorb them: it will
either treat them as the starting state of a new macro-task or
suggest committing/reverting based on the situation.
