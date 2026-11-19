# Migrating a project from v0.2 (OpenCode 1.x) to v0.3 (OpenCode 2.x)

This guide moves a project that uses the v0.2 templates (tag
[`v1-legacy`](../../tree/v1-legacy)) to the v0.3 templates. Do it at a
**macro-task boundary**: no macro-task `in-progress` or `wrap-up-pending`
in any service. Work on a dedicated branch.

Order matters: OpenCode first, then config, then prompts, then task files.

## 1. Upgrade OpenCode

Follow the official [V1 → V2 migration guide](https://opencode.ai/docs/migrate-v1).
In short:

1. Back up `~/.config/opencode` and `~/.local/share/opencode`.
2. Install OpenCode 2.x and check `opencode --version`.
3. Convert your **global** config: `agent` → `agents`, `disable` →
   `disabled`, model variants become `provider/model#variant`. Models
   stay in the global config; v0.3 never puts them in the repo.

OpenCode 2 reads v1 config in memory, but the v0.3 templates are native
v2: do not mix the two formats inside one `agents` entry.

> **CLAUDE.md**: OpenCode 2 does not read it. If any rule of yours lives
> in a `CLAUDE.md`, move it into the matching `AGENTS.md` now.

## 2. Add `sdd.config.yaml`

Copy `templates/sdd.config.yaml` to the monorepo root and set it to what
you already do. Two common shapes:

```yaml
# PR per macro-task into develop, CI on GitHub Actions
ci:  { provider: github, files: [] }
git: { remote: github, flow: git-flow, branch_per: task }
versioning: { enabled: true, current: "0.4" }
```

```yaml
# one branch per version shared by all services, releases tagged
ci:  { provider: none, files: [] }
git: { remote: github, flow: git-flow, branch_per: version }
versioning: { enabled: true, current: "0.9" }
```

## 3. Replace the configs

In `_project/` and in every service folder, replace `opencode.json` with
the v0.3 template, then re-apply what is really project-specific:

- extra MCP servers (move them under `mcp.servers`, with `type`)
- stricter rules you want to keep (append them at the end of
  `permissions`: the last matching rule wins)
- **drop** any `model` key (models belong in the global config) and any
  `instructions` inside an agent (it was never valid; only root
  `instructions` exists)

Check each workspace:

```bash
opencode debug config --standalone   # your file is listed, in v2 shape
opencode debug agents --standalone   # planner, builder, reviewers present; build/plan absent
```

## 4. Replace agents and skills

Copy `.opencode/agents/` and `.opencode/skills/` from the templates over
your workspaces. Keep your own additions: `<stack-name>` skills and
`<framework>-findings` skills are untouched by the templates.

## 5. Merge `AGENTS.md`

`AGENTS.md` mixes generic rules and project facts, so merge by hand:

- **keep yours**: the service description, Tech stack, Repository layout
  (source tree), project flow rules (`FR-*`), any domain sections;
- **take from the template**: Documentation map, Context partitioning,
  Agents, Workflow, Git workflow, the short Express/Free/Refactor sections,
  Documentation sync rules, File conventions, Boundaries;
- **add**: the `### Quality checks` list under Tech stack (mirror your CI
  if you have one).

A good shortcut: ask the `planner` (service) or `principal` (`_project`)
to do this merge in a free-mode session, with the template file as input.

## 6. Migrate the task files

New layout and naming are described in the `task-decomposition` skill.
For each service:

1. **Rename** each ledger `task-NN-<slug>.md` → `NN-<type>-<slug>.md`
   (`refactor-` tasks become type `ref`; pick `feat`, `fix`, `chore` or
   `spike` for the others). Keep `NN` and any letter (`12b`).
2. **Move** them into `v<x.y>/` folders if versioning is on (completed
   tasks into the version they shipped in; unscheduled ones into
   `backlog/`).
3. **Completed ledgers**: leave their content as it is — they are
   history and are not loaded again. Splitting them is optional.
4. **Not yet started ledgers**: nothing else to do.
5. **Rewrite `tasks.md`** to one line per macro-task (the template is in
   the skill). Long paragraphs from old `Completed` entries are already
   in the ledgers; drop them from the index.
6. **`task-log.md`**: keep the old entries; add one line
   `<date> — migration — task files renamed to v0.3 IDs` and use the
   one-line format from now on.

Old commits keep their old `Task:` footers. Do not rewrite history; the
`task-log.md` line above is enough to link old and new IDs.

## 7. First sessions

- Open `_project/` with OpenCode: the `principal` reads
  `sdd.config.yaml`, and its triage should be silent if everything is in
  place.
- Open each service: the `planner` triage should find no task in progress
  and propose the next macro-task. Its ID is the next free `NN`.
- If you use GitHub, ask the `principal` to set up branch protection
  (optional, `git-workflow` skill).
