# ADR-001: Business events as the backbone between services

- **Status:** Proposed
- **Date:** 2026-10-16
- **Deciders:** RSO project team
- **Related:** ADR-002, ADR-003; rules C1-C5, E1-E5, DB1-DB3, I4; project requirements #3, #18

## Context
Our app is 1 GUI + 6 microservices, each owned by one person and deployed independently on Kubernetes. In microservices every call crosses the network, which is less reliable than a method call, and a service can run on any node or region. If services call each other with chained request/response calls, one unresponsive service blocks its callers (coupling in time) and callers must know where the other service is (coupling in location).

Each service also owns its database (requirement #3), so no service can read another's tables. Services still need to know what happens elsewhere.

## Decision
Services integrate primarily through **business events** published to a message broker (broker choice: ADR-002).

- Events state **business facts** ("answer submitted", "reward granted"), not technical ones ("API call arrived").
- Services subscribe only to the channels they care about.
- Event payloads are agreed up front and kept stable. They are the shared contract between teams.
- An event carries the news, not every detail. A consumer that needs more calls the **API of the publishing service**.
- Use a **synchronous REST call (with timeout and retry)** only when the caller needs the result to continue.
- Use an **asynchronous command + `202 Accepted` + status resource** for long operations that can wait.
- Every service has its own database. Other services' data is reached only by API or events.

## Alternatives considered
- **Synchronous REST between all services.** Rejected: couples services in time and location, one slow or dead service stalls callers, and call chains multiply latency.
- **Shared database for all services.** Rejected: a distributed monolith. Schema changes break other services, services cannot be deployed or scaled independently, the database is a single point of failure, and teams must coordinate table layouts. It also violates requirement #3.

## Consequences
- **Positive:**
  - Services are decoupled in time and location. They are coupled only by the event format.
  - A new service can subscribe without changing publishers.
  - Each service can choose the database that suits it and scale on its own.
- **Negative / accepted risks:**
  - The system is **eventually consistent**. There are no ACID transactions across services.
  - Events can be delivered more than once, so every consumer must be idempotent (rule I4).
  - Event payloads become the new coupling point. Changing one affects every consumer, so evolution must be compatible (rule E5).
  - A request is harder to trace across services. Central logs, metrics and a correlation id are needed.
  - Publishing an event and writing to the service database is a dual write that can fail halfway (rule E4).
- **Follow-ups:**
  - Agree the payload of every event and the event envelope (event id, type, version, timestamp, `tenant_id`, correlation id) before development starts.
  - Decide the dual-write strategy (candidate ADR-004: transactional outbox).
  - Decide channel/subject naming.
  - Decide the multi-tenant isolation model (candidate ADR).

## Assumptions to validate
- The list of events can be fixed early enough for 6 people to build against it. Check after domain decomposition.
