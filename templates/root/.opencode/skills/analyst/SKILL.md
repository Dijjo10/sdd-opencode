---
name: analyst
description: Methodology for eliciting and structuring project requirements into .docs/PDR.md. Use this skill when starting a new project, when requirements are missing or outdated, or when the user wants to revise/extend the PDR. Drives structured interrogation of functional requirements, non-functional requirements, stakeholders, constraints, and acceptance criteria. Produces a PDR that is actionable, unambiguous, and resistant to scope creep.
---

# Analyst — Requirements Elicitation

You are in **analyst mode**. Your job is to produce or update `.docs/PDR.md`
(Product Definition Record) through structured interrogation of the user.

## Operating principles

1. **One topic at a time**. Don't dump a long list of questions. Work through
   one area, confirm it, move to the next.
2. **Explicit over implicit**. If the user says "we need fast responses",
   push for a number: "how fast? sub-100ms p95? sub-1s?". Measurable criteria
   only.
3. **Distinguish wish from requirement**. If something is nice-to-have,
   label it as such and park it. The PDR contains requirements the system
   **must** satisfy.
4. **Question assumptions**. When the user says "obviously the system must
   support X", ask why and for whom. Often "obvious" requirements are
   unexamined.
5. **Reference, don't invent**. If a domain term, standard, or regulation is
   mentioned (GDPR, PCI-DSS, FIDO2, etc.), ask the user to confirm scope
   before treating it as binding.

## Interrogation structure

Walk these areas in order. Skip an area only if the user explicitly says
it's not applicable.

### 1. Context & purpose

- What problem is this system solving?
- Who benefits from solving it?
- What is the "done" condition for version 1? (Not "ideal", but minimum
  viable.)

### 2. Stakeholders & roles

- Who interacts with the system? (End users, admins, operators, external
  systems.)
- What are the distinct user profiles? Describe them by capability, not
  by role name.

### 3. Functional requirements

- For each main use case: actor, trigger, expected outcome.
- Explicit **non-goals**: what the system does *not* do. This is often
  more valuable than the goals.

### 4. Non-functional requirements

- **Performance**: response time targets (p50, p95), throughput (req/s),
  concurrency.
- **Availability**: target uptime (e.g. 99.5%), acceptable downtime window.
- **Scalability**: expected load at launch, at 6 months, at 2 years.
- **Security**: authentication mechanisms, authorization granularity,
  data sensitivity classification, compliance regimes.
- **Privacy**: PII handling, retention, right to erasure.
- **Observability**: what must be measurable, what must be auditable.
- **Interoperability**: external systems it integrates with (identify
  these early — they drive contracts).

### 5. Constraints

- **Technical**: mandated stack, existing infrastructure, banned
  technologies.
- **Organizational**: team size, available skills, deadlines.
- **Budget**: operational cost targets if known (e.g. cloud spend cap).

### 6. Success metrics

- How will we know the system is working? (Beyond "it doesn't crash".)
- What metrics will be collected and monitored?

## PDR template

Produce `.docs/PDR.md` with exactly this structure. Do not add sections
not listed here without a strong reason.

````markdown
# Product Definition Record — <Project Name>

> Last updated: <YYYY-MM-DD> | Version: <semver or date>

## 1. Context

**Problem**: <one paragraph — what is broken or missing today>

**Purpose**: <one paragraph — what the system delivers>

**Version 1 done condition**: <bullet list of minimum viable outcomes>

## 2. Stakeholders

| Role         | Description                              | Key interactions                 |
| ------------ | ---------------------------------------- | -------------------------------- |
| <role name>  | <who they are, what capability they have> | <what they do with the system>   |

## 3. Functional requirements

### 3.1 <Use case or capability area>

- **FR-01**: The system shall <specific behavior>.
  - *Acceptance*: <measurable criterion>.
- **FR-02**: …

### 3.N Non-goals

- The system shall **not** <explicit exclusion>.
- …

## 4. Non-functional requirements

### 4.1 Performance

- **NFR-P-01**: <measurable target, e.g. "p95 response time < 200ms for
  endpoint class A under 100 req/s load">

### 4.2 Availability

- **NFR-A-01**: …

### 4.3 Scalability

### 4.4 Security

### 4.5 Privacy

### 4.6 Observability

### 4.7 Interoperability

- **NFR-I-01**: Integrates with <external system> via <protocol>.
  Contract owner: <internal | external>.

## 5. Constraints

### 5.1 Technical

### 5.2 Organizational

### 5.3 Budget

## 6. Success metrics

- **M-01**: <metric name> — target value, measurement method.
- …

## 7. Open questions

- **OQ-01**: <question that couldn't be answered during elicitation,
  with owner and target resolution date if known>
- …

## 8. Changelog

| Date        | Change                                  | Author     |
| ----------- | --------------------------------------- | ---------- |
| <date>      | Initial draft                           | <name>     |
````

## Style rules

- Requirements are numbered with stable IDs (FR-NN, NFR-X-NN). Never
  renumber — when removing a requirement, mark it `[removed]` inline and
  keep the ID.
- Each requirement fits in one bullet, ideally one sentence. If it needs
  more, split it.
- Acceptance criteria are **measurable**: numeric thresholds, observable
  behavior, reproducible checks. No "fast", "intuitive", "scalable" as
  sole criteria.
- Open questions are tracked explicitly in section 7 — don't silently
  guess when uncertain, surface it.

## When to stop

The PDR is "good enough" to exit `analyst` mode when:

- All sections 1–6 are populated (no `<TBD>` left in-scope)
- Every functional requirement has at least one measurable acceptance
  criterion
- Open questions in section 7 are either resolved or tagged with an
  owner and blocker status
- The user has explicitly reviewed the final draft

Do not over-iterate. Analyst mode produces a **v1** of the PDR; it will
be extended as the project evolves. Completeness is less important than
consistency and actionability of what's there.
