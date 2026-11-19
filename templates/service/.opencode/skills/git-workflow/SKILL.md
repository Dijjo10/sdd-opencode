---
name: git-workflow
description: Branching, integration and release procedure of the SDD pattern, driven by the `git` and `ci` sections of `sdd.config.yaml`. Load it when a macro-task starts (create its branch), at the end of wrap-up (push, pull request, CI, merge), when a version is closed (changelog, tag), or when setting up branch protection.
---

# Git workflow

This skill turns the `git`, `ci` and `versioning` settings of
`sdd.config.yaml` (monorepo root) into concrete git operations. Read the
file first. If it is missing, use the defaults: `git.remote: none`,
`ci.provider: none`, `versioning.enabled: false`.

The permission policy in `opencode.json` already asks the user before
`git push`, `git merge`, `gh pr merge`, `gh release`, `git reset --hard`
and `git clean`, and denies force-push. Never try to work around an
`ask` or a `deny`.

## Shared-tree rules (always)

Several services often work in the **same checkout** at the same time.
These rules keep them from damaging each other's work:

- Touch only your own workspace folder. A needed change outside it is an
  Emergent decision flagged to `_project`.
- Commit with **explicit paths**: `git add -- <path...>` (needed for new
  files) then `git commit -m "<msg>" -- <path...>`.
  Never `git add -A`, `git add .` or a bare `git commit -a`.
- During a micro-task, **never** run `checkout`/`switch`, `merge`,
  `rebase`, `stash`, or a global `reset`/`restore`/`clean`. Branch
  operations happen only at the points described below, and only when
  no other agent is working in the same checkout.
- One builder per service at a time; parallel work is only across
  different services.
- `index.lock` exists → wait and retry. Never delete it by hand.

## Names

| Thing | Pattern | Example |
| ----- | ------- | ------- |
| Integration branch | `main` (github-flow) or `develop` (git-flow) | `develop` |
| Macro-task branch (`branch_per: task`) | `<type>/<task-id>` | `feat/v0.2-03-auth`, `fix/07a-token-refresh` |
| Version branch (`branch_per: version`) | `version/<major>.<minor>` | `version/0.2` |
| Tag | `v<major>.<minor>.<patch>` | `v0.2.0` |

`<type>` and `<task-id>` come from the macro-task ID (see the
`task-decomposition` skill). Record the branch name in the macro-task
entry of `tasks.md`.

## 1. Start of a macro-task

Run this when a macro-task moves to `in-progress` (planning mode), before
the planning-docs commit.

- `git.remote: none` and the user works on a single branch → stay on the
  current branch; nothing to do.
- `branch_per: version` → make sure the version branch exists
  (`git switch -c version/<x.y> <integration>` the first time, then stay
  on it). All services share it.
- `branch_per: task`:
  1. Check that **no other service has uncommitted work** in this
     checkout (`git status --porcelain`, look outside your folder). If
     another service is busy, do not switch: propose a worktree instead
     (`git worktree add ../<repo>-<service> -b <branch> <integration>`)
     and stop until the user decides.
  2. Update the integration branch if a remote exists
     (`git fetch origin <integration>`).
  3. `git switch -c <type>/<task-id> origin/<integration>` (or the local
     integration branch when `git.remote: none`).

## 2. End of wrap-up — integration

Run this after the wrap-up commit. Each step that leaves the machine is
an `ask` in the permission policy: the user approves it.

| `git.remote` | `branch_per: task` | `branch_per: version` |
| ------------ | ------------------ | --------------------- |
| `github` | push → PR → CI → merge (below) | push the version branch; PR happens at version close |
| `other` | push the branch; tell the user to merge on the forge | push the version branch |
| `none` | local merge on approval: `git switch <integration> && git merge --no-ff <branch>` | nothing until version close |

GitHub, `branch_per: task`:

1. `git push -u origin <branch>`
2. `gh pr create --base <integration> --head <branch> --title "<type>(<scope>): <macro-task title>" --body-file <tmp>` —
   the body is the ledger's **Summary** plus a link to the ledger.
3. If `ci.provider` is not `none`: `gh pr checks <branch> --watch`. A
   failing check **blocks the integration**: report it, propose a fix
   micro-task, stop. Never merge red.
4. Ask the user to approve the merge, then
   `gh pr merge <branch> --merge --delete-branch`. Use a merge commit
   (not squash): micro-task commits stay visible for audit.
5. `git switch <integration> && git pull --ff-only`.

## 3. Version close (`versioning.enabled: true`)

Owned by `principal` in `_project`. A version is closed when every
macro-task in `.docs/tasks/v<x.y>/` of every service is `done` or
`skipped`.

1. Write the `CHANGELOG.md` entry for the version from the macro-task
   Summaries (see `task-decomposition`).
2. Bring the version to its release branch:
   - `branch_per: version` → PR (or local merge) `version/<x.y>` →
     integration branch, same steps as section 2.
   - `flow: git-flow` → PR (or local merge) `develop` → `main`.
3. Tag on the release branch (`main`):
   `git tag -a v<x.y>.0 -m "v<x.y>.0"` and `git push origin v<x.y>.0`.
4. Bump `versioning.current` in `sdd.config.yaml` to the next version.

## 4. Branch protection (GitHub, once)

Owned by `principal`, at project setup, only on explicit user request.
Protect the integration branch (and `main` under git-flow): pull request
required, no force-push, no deletion, and required status checks when CI
is enabled. Show the command and wait for approval:

```
gh api -X PUT repos/<owner>/<repo>/branches/<branch>/protection \
  -F required_pull_request_reviews=null \
  -F enforce_admins=false \
  -F restrictions=null \
  -F 'required_status_checks[strict]=true' \
  -f 'required_status_checks[contexts][]=<check name>' \
  -F allow_force_pushes=false -F allow_deletions=false
```

Without CI, set `required_status_checks=null`. A solo developer usually
keeps `required_pull_request_reviews=null`: the PR is the integration
point, not a second reviewer.
