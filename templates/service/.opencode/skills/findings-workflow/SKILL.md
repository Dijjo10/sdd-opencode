---
description: Methodology for capturing and reusing experiential findings about a specific framework or library — recurring errors, gotchas, and discoveries that are NOT in the official docs. Invoked by the builder when it hits or discovers a framework/library issue, and when promoting a recurring finding into a per-framework findings skill. Distinct from curated stack skills: this governs the experiential, auto-accumulated knowledge layer.
---

# findings-workflow

## What this is

**Experiential findings** are the tribal knowledge you only learn by hitting
it: version-specific bugs, undocumented behavior, surprising interactions
between libraries, workarounds that aren't obvious. This is distinct from:

- the **curated `<stack-name>` skill** — the conventions *you* decided;
  authoritative
- **Context7** — the provider's official live docs; authoritative

Findings capture what neither of those contains. **On conflict, the curated
skill plus Context7 win** — findings are experiential and can age (a fix may
be obsoleted by a dependency upgrade).

This workflow is for **framework/library** findings only. Process/flow
lessons (e.g. "the mockup wasn't loaded before UI work") are **not** findings
— route them per the 3-way taxonomy in `AGENTS.md` (**Project flow rules**,
or a universal pattern rule that belongs in the agent prompts).

## Two levels

1. **Raw log** — `.docs/findings/<framework>.md`, append-only, low friction.
   Every finding lands here first. **Not loaded by default** — it can grow
   large; you consult it on demand (grep) when you hit an issue.
2. **Findings skill** — `.opencode/skills/<framework>-findings/SKILL.md`.
   Holds only **promoted** (recurring) findings. **Loaded when working on
   `<framework>`**, alongside the curated `<stack-name>` skill.

The point of the split: the raw log is cheap to write and never costs context
until you look; the findings skill is small, curated-by-recurrence, and worth
carrying in context every time you touch that framework.

## Finding schema (raw log)

````markdown
### F-<NN> — <short symptom / title>
- **Trigger**: <what surfaces it — error message, symptom, failing test>
- **Cause**: <root cause, one or two lines>
- **Fix / convention**: <the resolution or the rule to follow>
- **Verified**: <yes — tests pass | how it was confirmed>
- **Hits**: <count>          # bump on each recurrence
- **First seen**: <task/micro-task ref, date>
- **Promoted**: <no | yes → moved to <framework>-findings on <date>>
````

## Lifecycle (builder-owned)

- **On hitting or discovering** a framework/library issue: search
  `.docs/findings/<framework>.md`. If the finding is absent, append it with
  `Hits: 1`. If present, **increment `Hits`** and refine the fix if you
  learned more.
- **Promotion at 2 hits**: when a finding reaches `Hits: 2`, copy it into the
  `<framework>-findings` skill (create that skill from the scaffold below if
  it doesn't exist yet) and set `Promoted: yes` in the raw log. Promoted
  findings are the ones loaded on every future session for that framework, so
  you stop re-hitting them.
- **Only promote verified findings** — the fix must have made the tests pass.
  A guessed or unconfirmed fix is not a finding; leave it in the raw log with
  `Verified: no` until confirmed.

## Loading

Load `<framework>-findings` **before writing `<framework>` code** (if it
exists), next to the curated `<stack-name>` skill. Load **selectively** —
only the framework the current micro-task touches, never all of them, so a
multi-stack service doesn't pay for findings it isn't using.

## Maintenance (anti-bloat)

Periodically consolidate a `<framework>-findings` skill: merge near-duplicate
findings, and drop entries obsoleted by a dependency upgrade. Run this on
request, not automatically. If a findings skill grows unwieldy, that's a
signal the curated `<stack-name>` skill should absorb the stable ones as real
conventions.

## The `<framework>-findings` skill — scaffold

When you first promote a finding, create
`.opencode/skills/<framework>-findings/SKILL.md` with this shape:

````markdown
---
description: Experiential findings for <framework> — recurring, verified gotchas and conventions discovered on the job. Load before writing <framework> code. Complements (never overrides) the curated <stack-name> skill and Context7.
---

# <framework>-findings

> Promoted findings (Hits >= 2), newest at the bottom. These are experiential:
> if one conflicts with the curated `<stack-name>` skill or Context7, those
> win. Keep entries verified and concise.

### F-<NN> — <title>
- **Trigger**: <symptom>
- **Cause**: <root cause>
- **Fix / convention**: <resolution or rule>
- **Verified**: yes
- **First seen**: <task ref>
````
