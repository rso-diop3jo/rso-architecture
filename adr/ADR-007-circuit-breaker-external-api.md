# ADR-007: Circuit breaker, timeout and retry on every call to the external API

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-003, ADR-005, ADR-006; rules C1, C4, I3; project requirements #9 (external API), #13 (isolation and fault tolerance)

## Context
- Requirement #9 (3 points) needs at least one external API used inside one of our microservices, with proper authentication.
- An external API is outside our control. It can be slow, down, or rate-limiting us, and we can neither fix it nor see why.
- Rule C1 says a synchronous call needs a timeout and retry. Rule C4 warns that a caller blocks when the service it calls stops responding.
- Requirement #13 (5 points) asks for fault-tolerance mechanisms such as Circuit Breaker and Retry, a demonstration that the system keeps working when one service fails, and an explanation of how isolation limits the damage. The team decided to cover #13 with a circuit breaker plus retry and not to build other isolation mechanisms.
- Services integrate mainly through events (ADR-001), so synchronous calls are few. The external API is the clearest place where one is unavoidable.
- Go is the default service language (ADR-005), and the chosen library is a Go library.

## Decision
**Every outgoing call to the external API goes through a circuit breaker ([sony/gobreaker](https://github.com/sony/gobreaker) v2), with a client-side timeout and a retry with backoff.**

Key details:

- One breaker per external dependency (per external host), not one global breaker.
- The timeout is set on the HTTP client. gobreaker does not time calls out: its `Timeout` setting is how long the breaker stays open before it probes again.
- Counted as failures: 5xx responses, timeouts and network errors. Not counted: 4xx responses (our request was wrong, the API is healthy), set with `IsSuccessful`. Context cancellations are excluded with `IsExcluded`.
- Retry wraps the breaker: each attempt goes through the breaker, and retrying stops when it returns `ErrOpenState`.
- When the breaker is open, the service degrades instead of failing hard. TODO (user): what the traveler sees then, defined in the use case of the service that calls the API.
- State changes (`OnStateChange`) are written to stdout as logs (rule R2) and exposed as a metric (rule R3).
- Breaker state lives in the memory of each pod; replicas do not share it.

Out of scope: publishing to NATS. A broker outage is handled by retry and an outbox (ADR-003, candidate ADR-004), not by a breaker.

## Alternatives considered
- **Timeout and retry only, no breaker.** Rejected: during an outage every request still waits the full timeout times the number of retries, and the retries add load to a dependency that is already failing. Requirement #13 also names the circuit breaker.
- **Breaker in the infrastructure (service mesh or gateway, for example Istio).** Rejected: needs a mesh we have not decided to run, and a breaker outside the code is harder to show in the demo and in code review.
- **Own hand-written breaker.** Rejected: the three-state logic (closed, open, half-open) with counting is easy to get subtly wrong, and gobreaker already implements it in a small library.
- **Also wrapping every service-to-service call.** Not decided here: see follow-ups.

## Consequences
- **Positive:**
  - A dead external API costs one fast failure instead of a full timeout per request.
  - The failing API gets less traffic while it recovers.
  - Gives the circuit breaker and retry mechanisms for requirement #13 (5 points), visible in the code.
- **Negative / accepted risks:**
  - Thresholds need tuning. Too sensitive and a short blip cuts the feature off, too lax and the breaker never protects. gobreaker's defaults are: open after more than 5 consecutive failures, stay open 60 seconds.
  - State is per pod. With several replicas each one trips separately, and the API still gets traffic from the others.
  - A degraded path must be designed, built and tested. An open breaker with no fallback is only a faster error.
  - gobreaker is Go-only. A service in another language (allowed by ADR-005) needs an equivalent library with the same rules.
- **Follow-ups:**
  - TODO (user): decide who owns this point (tracked in `_ToDo`).
  - Decide which service calls the external API, after service boundaries are defined.
  - Define the degraded behaviour in that service's use case.
  - Decide whether the breaker also wraps synchronous service-to-service calls (for example the gRPC link in ADR-006). It would let us show "one of our services fails" for #13 more convincingly.
  - Write the short explanation of how service isolation limits the impact of failures (requirement #13) in the documentation.
  - Once accepted, add a rule to `architecture/rules.md`: calls to external APIs use breaker, timeout and retry.

## Assumptions to validate
- A demo where the external API fails (for example a test server returning errors) is enough to show #13's "system keeps working when one service fails". The PDF wording speaks about a service of ours. Ask at the next vaje.
- Breaker plus retry plus a documented explanation earns the points of #13 without extra isolation mechanisms. Not confirmed.
- gobreaker v2 behaves as its README describes (checked 2026-10-22: generic `NewCircuitBreaker[T]`, the settings named above, in-process state). Verify in a small spike against a failing test server.
