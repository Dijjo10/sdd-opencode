---
name: architect-project
description: Methodology for designing cross-service architecture from PDR requirements. Use this skill after PDR is drafted, when defining or revising how the system decomposes into microservices, how services communicate, where boundaries lie, and what infrastructure supports them. Produces .docs/architecture.md, .docs/infra.md, and OpenAPI contracts in .docs/contracts/. Applies bounded-context and single-responsibility principles with explicit tradeoff rationale.
---

# Architect-Project — Cross-Service Architecture

You are in **architect-project mode**. Your job is to produce or update
the project-level architecture artifacts: `.docs/architecture.md`,
`.docs/infra.md`, and `.docs/contracts/openapi-*.yaml`.

## Operating principles

1. **Derive from PDR, don't invent**. Every architectural choice must
   trace to a requirement (functional or non-functional) in `.docs/PDR.md`.
   If a choice has no requirement behind it, question whether it's needed.
2. **Bounded contexts first**. Decompose by domain boundary, not by
   technical layer. A "user service" and an "auth service" are
   suspicious — they likely share a bounded context.
3. **Explicit coupling**. Every inter-service communication is a contract.
   Sync (REST/gRPC) and async (events) have different failure modes —
   choose deliberately.
4. **Document tradeoffs, not just decisions**. When choosing pattern A
   over B, state what was lost. Future readers need to know what was
   considered.
5. **Infrastructure is part of architecture**. A microservice topology
   that ignores deployment reality (where things run, how they're
   networked, how they fail) is incomplete.

## Decomposition methodology

Walk these steps in order.

### 1. Identify bounded contexts

From the PDR, extract the distinct domains. Ask:

- Does this area have its own vocabulary? (Different meanings for the
  same term across areas = context boundary.)
- Does this area evolve independently? (Different change cadence =
  separation signal.)
- Does this area have its own data that shouldn't leak? (Data ownership
  = context boundary.)

A bounded context **may** become a service. Not every context has to —
small contexts can merge if they share lifecycle.

### 2. Candidate services

For each candidate service, specify:

- **Owns**: the data and behavior it's authoritative for
- **Consumes**: what it needs from others
- **Exposes**: what others need from it
- **Does not**: explicit non-responsibilities

If "exposes" is empty, it's probably not a service — it's a library.

### 3. Communication patterns

For each pair of services that interact, decide:

- **Sync or async?** Sync if the caller needs the result to proceed.
  Async if the caller only needs the event to eventually be processed.
- **If sync**: REST, gRPC, GraphQL? Consider tooling, language
  ecosystem, versioning strategy.
- **If async**: choose a message broker, define topic/queue naming
  convention.
- **Failure semantics**: at-most-once, at-least-once, exactly-once?
  Retries, DLQ, idempotency.

### 4. Cross-cutting concerns

- **Authentication/Authorization**: centralized (gateway/identity
  service) or distributed? Token format (JWT, Paseto)? Validation
  strategy.
- **Observability**: log aggregation, metrics, tracing. Where do they
  converge?
- **Configuration**: secrets management, feature flags, env
  propagation.

### 5. Infrastructure

- Target runtime: VMs, containers, orchestrator.
- Network topology: public ingress, internal, isolation boundaries.
- Persistence: per-service databases, shared caches, event store.
- CI/CD: build/test/deploy pipeline per service.

## Output artifacts

### `.docs/architecture.md` — template

````markdown
# Project Architecture — <Project Name>

> Last updated: <YYYY-MM-DD>
> Related: [PDR](./PDR.md) | [Infra](./infra.md) | [Contracts](./contracts/)

## 1. System overview

<One paragraph: what the system does, at architecture level. Not the
product pitch — the technical synopsis.>

## 2. Service topology

### 2.1 Service inventory

| Service        | Owns                          | Exposes                 | Consumes from           |
| -------------- | ----------------------------- | ----------------------- | ----------------------- |
| `<ms-name>`    | <domain data/behavior>        | <endpoints/events>      | <other services>        |

### 2.2 Service diagram

```
<ASCII diagram or reference to external diagram file>
```

## 3. Communication

### 3.1 Sync interactions

| Caller         | Callee         | Protocol   | Contract                          | Auth          |
| -------------- | -------------- | ---------- | --------------------------------- | ------------- |
| `<ms-a>`       | `<ms-b>`       | REST       | `contracts/openapi-<ms-b>.yaml`   | JWT (bearer)  |

### 3.2 Async interactions

| Producer       | Consumer(s)    | Broker     | Topic/Queue            | Delivery      |
| -------------- | -------------- | ---------- | ---------------------- | ------------- |
| `<service-a>`  | `<service-c>`  | <broker>   | `<domain>.<event>.v1`  | at-least-once |

## 4. Cross-cutting concerns

### 4.1 Authentication & authorization

<Scheme: centralized/distributed. Token format. Validation. Session
lifecycle if applicable.>

### 4.2 Observability

<Logging format, aggregation endpoint. Metrics stack. Tracing propagation.>

### 4.3 Configuration & secrets

<Where config lives, how it's injected, how secrets are managed.>

## 5. Architectural decisions

### ADR-001 — <Short decision title>

- **Context**: <1–3 sentences>
- **Decision**: <what was decided>
- **Alternatives considered**: <what else, and why not>
- **Consequences**: <what changes as a result, both good and bad>

### ADR-002 — …

## 6. Changelog

| Date        | Change                                  | Author     |
| ----------- | --------------------------------------- | ---------- |
````

### `.docs/infra.md` — template

````markdown
# Project Infrastructure — <Project Name>

> Last updated: <YYYY-MM-DD>

## 1. Deployment target

- **Platform**: <e.g. self-hosted Proxmox + k3s, AWS ECS, bare VMs>
- **Regions / zones**: <geographical and failure-domain considerations>

## 2. Environments

| Environment | Purpose                   | Data          | Access                     |
| ----------- | ------------------------- | ------------- | -------------------------- |
| dev         | Local development         | synthetic     | developers                 |
| staging     | Pre-prod validation       | anonymized    | dev team + QA              |
| prod        | Production                | real          | restricted                 |

## 3. Network topology

<Describe ingress, internal network, egress rules, isolation boundaries.>

## 4. Persistence layer

| Service        | Store type     | Instance                  | Backup policy          |
| -------------- | -------------- | ------------------------- | ---------------------- |
| `<service-a>`  | <store kind>   | <topology>                | <retention policy>     |

## 5. Shared infrastructure

- **Message broker**: <kind, topology, sizing — only if used>
- **Cache**: <kind, topology — only if shared across services>
- **Object storage**: <kind, location — only if used>
- **Secrets management**: <approach: vault service, encrypted files, sealed-secrets, KMS, etc.>

## 6. CI/CD

### 6.1 Pipeline per service

<Describe stages: build, test, scan, package, deploy. Where it runs.>

### 6.2 Quality gates

<Static analysis, security scans, integration tests before deploy. Specify which tools and at which stage.>

### 6.3 Deployment strategy

<Blue/green, canary, rolling. Rollback procedure.>

## 7. Observability stack

- **Logs**: <aggregator, format, retention>
- **Metrics**: <Prometheus, Grafana, dashboards>
- **Tracing**: <OpenTelemetry pipeline>
- **Alerting**: <alertmanager, notification channels>

## 8. Changelog
````

### `.docs/contracts/openapi-<ms-name>.yaml` — conventions

For each service that exposes a sync API, produce one OpenAPI file.
Conventions:

- **Filename**: `openapi-<ms-name>.yaml`
- **Version in `info.version`** follows semver; bump per change category
  (major: breaking, minor: additive, patch: doc/description)
- **Base path convention**: `/<ms-name>/v<major>/...`
- **Auth schemes** declared in `components.securitySchemes` and
  referenced per endpoint — never leave auth implicit
- **Error responses**: every endpoint documents 4xx/5xx bodies; use a
  shared error schema referenced from `components.schemas`
- **Examples**: every request/response schema has at least one example

Do not attempt a full OpenAPI spec from scratch — start from the caller's
needs (what data does the other service need?) and fill in progressively.

## Style rules

- **ADRs are small**: one decision each, 5–10 lines total. If a decision
  needs more explanation, the decision itself is probably too broad —
  split it.
- **Tables over prose** for comparative information (services, endpoints,
  interactions). Prose for narrative context only.
- **No bikeshedding**. Framework choices, naming conventions, and similar
  decisions get an ADR only if they have non-trivial consequences.
- **Cross-reference, don't duplicate**. If infrastructure details belong
  in `infra.md`, reference them from `architecture.md`, don't copy.

## When to stop

Architecture v1 is complete when:

- Every service in the inventory has `Owns / Exposes / Consumes` filled
- Every sync interaction has a corresponding OpenAPI file (even if
  skeletal)
- Every async interaction has topic/queue name and delivery semantics
- ADRs cover the non-obvious choices (not every choice, just the ones
  with tradeoffs worth recording)
- The user has reviewed and agreed

Expect architecture to evolve. The goal of v1 is to enable implementation
to start, not to foresee everything.
