# Architecture rules (RSO project)

Distilled from the lecture notes (`CONSOLTING_AGENT_raw_text.md`) and the project requirements
(`[SLO] Navodila za projekt in kriterij ocenjevanja.pdf`). The architect agent checks every design,
ADR, spec and schema against this file. Edit this file when a rule changes; the agent reads it at the start of every session.

Legend: **[L]** = stated in the lecture. **[V]** = stated in a practical session (vaje), 2026-10-22. **[C]** = correction or clarification added after fact-checking.

## 1. Decomposition
- **D1 [L]** Test every service boundary: "Can one team change, test and deploy this service without coordinating with other teams?" If not, the boundary is wrong.
- **D2 [L]** Learn the domain before drawing boundaries. Better domain knowledge gives better boundaries.
- **D3 [L]** Before building anything, ask whether the problem is already solved (identity: Keycloak; payments: Stripe; email: SendGrid). Do not build generic services.
- **D4 [L]** Granularity for this project: 1 GUI + 6 microservices (the GUI is not a microservice). One repository per service.
- **D5 [L]** Write an ADR for every key decision (`adr/ADR-NNN-*.md`): what was decided and why.

## 2. Communication: API or event?
- **C1 [L]** Caller needs the result to continue -> synchronous API call (REST/gRPC) **with timeout and retry**.
- **C2 [L]** A business fact other services care about -> event (NATS JetStream; see ADR-002).
- **C3 [L]** Long operation that can wait -> asynchronous command to a queue + `202 Accepted` + a status resource.
- **C4 [L]** Never couple services with chains of request/response calls: when one stops responding, the caller blocks.
- **C5 [L]** Avoid any architecture that is coupled in time or location. Format coupling (the event payload) is the only accepted coupling.

## 3. REST API design
- **A1 [L]** Version in the URL (`/v1/...`, `/v2/...`) so breaking changes do not break legacy clients.
- **A2 [L]** Batch endpoints (array of items) instead of one call per item. Optimise further from logs and metrics, not by guessing.
- **A3 [L]** Intent-oriented endpoints, not CRUD over the database: `POST /orders`, `POST /orders/42/cancel`, `GET /orders/42`. Business rules stay in the service.
- **A4 [L]** Separate endpoint classes with different needs: `client-api` (secure), `service-api` (high throughput, internal). Each may use a different technology.
- **A5 [L]** Document REST APIs with OpenAPI 3.1 (OAS 3.1).
- **A6 [L]** Use the standardised REST error (fault) format for all errors. [C] That standard is RFC 9457 "Problem Details for HTTP APIs".

## 4. Idempotency
- **I1 [L]** READ is idempotent by design. WRITE/UPDATE is not, so it needs a unique key so the server knows whether it already executed.
- **I2 [C]** The lecture notes say "e-tag". That is wrong for POST: use an `Idempotency-Key` header. `ETag` / `If-Match` is for optimistic concurrency on updates, not for idempotency.
- **I3 [L]** Retry + timeout is the standard way to make sure calls complete.
- **I4 [C]** Event delivery is at-least-once (JetStream), so duplicates will happen. Every event consumer must be idempotent.

## 5. Events
- **E1 [L]** Events carry **business** facts ("article X bought 2x", "item Y returned", "new product added"), never technical facts ("API call X arrived", "file Z written").
- **E2 [L]** Agree the **payload of every event** at the start of the project. Payloads must stay stable. APIs can be versioned; events are our new coupling.
- **E3 [L]** If a consumer needs more than the event carries, it calls the **API of the service that published the event**. The event is the news; the API gives the detail.
- **E4 [L]** A service writes to its own database and publishes the event as part of the same operation. [C] Doing both reliably is the dual-write problem; the design must say how (e.g. transactional outbox).
- **E5 [L]** Compatibility over time: schema evolution with expand -> migrate -> contract.

## 6. Data
- **DB1 [L]** One database per service. **Never a shared database** between services (that is a distributed monolith). This is also mandatory project requirement #3.
- **DB2 [L]** Other services' data is reached only via API or events.
- **DB3 [L]** Accept eventual consistency. No ACID transactions across services.
- **DB4 [L]** Choose the database that fits each service (e.g. catalogue: search/document/vector store; purchases: relational for integrity).

## 7. Multi-tenancy
- **T1 [L]** The project is multi-tenant. Carry `tenant_id` through the entire chain from day one.
- **T2 [L]** Chain: JWT with `tenant_id` -> API gateway (per-tenant quotas, routing) -> service (tenant context in every call) -> events (header or subject; NATS accounts / Kafka ACLs) -> data (filtering or RLS; schema- or DB-per-tenant) -> Kubernetes (namespace, ResourceQuota, NetworkPolicy).
- **T3 [L]** Data isolation options: shared DB (least secure), schema per tenant, database per tenant (most secure). Pick one by ADR; the choice affects DevOps and CI/CD.
- **T4 [L]** One wrongly configured filter exposes one tenant's data to another. Every design must say where tenant filtering is enforced.

## 8. Runtime and operations
- **R1 [L]** async/await, not thread-per-request (about 1 MB per thread). Reactive only if data streaming is needed.
- **R2 [L]** Services never write system logs into the container (read-only, lost when the container dies). Emit logs to stdout and collect centrally (Grafana Loki or Fluentd).
- **R3 [L]** Numeric metrics: Prometheus via `/metrics`, shown in Grafana. Collect RED (rate, errors, duration p50/p99) for services, USE (utilisation, saturation, errors) for resources, and the four Google SRE golden signals.
- **R4 [L]** Health checks exposed per service for Kubernetes: liveness, readiness, startup.

## 9. Service structure (clean architecture)
Graded under mandatory requirement #2 ("Koncepti clean architecture"). Decision record: ADR-008.
- **S1 [V]** Every service is built as an "onion", from the inside out: **domain** (the elementary classes and functions, e.g. the classes that define the data the service keeps), **use cases** (everything the service offers and does), then the outer ring with two parts: **web** (the service's API, incoming traffic) and **adapters** (outgoing traffic, e.g. calls to other services). In the repository these are four folders: `domain`, `usecases`, `web`, `adapters`.
- **S2 [V]** Dependencies point inward only. An outer ring may use inner rings, never the reverse: web uses use cases, use cases use the domain, the domain imports nothing from the rings around it.
- **S3 [V]** Services talk to each other only through another service's public API (the part exposed to other services and to the outside) or through events. Never import a class or package from another service.
- **S4 [C]** S2 and "use cases need a database or another service" only fit together if the use-case ring defines what it needs (an interface) and an adapter implements it. Keywords: ports and adapters, dependency inversion.
- **S5 [V]** Storage sits behind an adapter, so the implementation can differ per environment: in-memory in tests and in pipelines that only check that deployment works (fast, no database needed), a real external database in staging and production. The choice is configuration (see P4), not a code change.

## 10. Delivery (CI/CD, Helm, secrets)
Optional requirements #6 (CI/CD) and #7 (Helm charts) are in scope.
- **P1 [V]** One repository = one pipeline = one service. A change builds, tests and deploys only that service.
- **P2 [V]** Order the jobs with `needs`: test -> build (image, pushed to a registry) -> deploy-staging. A deploy job must depend on the build job, otherwise there may be no image to pull. The example pipeline shown at the vaje has `deploy-prod` with `needs: test` only: it can run before the image exists and then fails. Do not copy that.
- **P3 [V]** Build the image once, then promote it. Production deploy is manual (a button) and pulls the image that already passed the tests and was checked in staging. It does not build again.
- **P4 [V]** Environment differences live in Helm values: one values file per environment (dev, staging, production, and a test setup using in-memory storage). They hold e.g. the database address. Not in code.
- **P5 [V]** Secrets (Azure credentials, Docker Hub token, cluster name and resource group) are GitHub Actions secrets, never in the repository. The cluster name counts as a secret too: with leaked credentials someone could deploy to our cluster. Name the secrets so Helm and the workflow can read them.

## 11. Principles to cite in reviews
Idempotency (retries, idempotent consumer, idempotency key) - Declarative design and convergence (state from events, configuration as data) - Loose coupling (events, DB per service, tenant context) - Compatibility over time (schema evolution, expand-migrate-contract).

## 12. Grading (read the PDF for the full table)
All mandatory requirements must be met, plus at least 50% of the total points. Mandatory: repository, microservices on Kubernetes, DB per service, documentation, cloud deployment, GUI. Final grade = alpha * code + (1 - alpha) * final presentation + interim presentation. Tie a finding to a requirement number when it affects points.

## Open / undecided
- Tenant isolation model (T3): undecided.
- Dual-write strategy (E4): undecided (candidate ADR-004).
- Event envelope and subject naming: undecided.
- What a tenant is in our app (the vaje example: one faculty, or one group of friends). T3 depends on it.
- Which secrets live at organization level and which per repository (P5).
- Where the databases live and who pays for them (the vaje idea: databases on one person's Azure account, the cluster on another's). Not decided.
- Non-blocking flow from the GUI: the vaje suggested the UI publishes and subscribes to events, while requirement #15 and ADR-006 say the GUI talks REST. How does the GUI learn the result without blocking (rule C3)?
