# ADR-005: Go as the default service language (per-service deviation allowed)

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-002; rules A5, R2, R3, R4; project requirements #2 (microservices, cloud-native), #6 (CI/CD); DEADLINE 1 "Ogrodje in razvojno okolje"

## Context
Each of the 6 services is owned by one person and lives in its own repository (ADR-001). Team members have different language preferences and expertise (some prefer TypeScript). The course assistant suggested Java (KumuluzEE or Spring Boot).

DEADLINE 1 asks us to name our frameworks and development environment and lists KumuluzEE, Spring Boot, Node.js, Docker and Kubernetes only as examples. The project PDF names no required programming language (checked 2026-10-22: no mention of Java, KumuluzEE, Spring or Node.js).

The project is small and exists to learn cloud deployment and architecture design. Cloud-native concepts are already a steep learning curve, so we want as little extra friction from the language and its tooling as possible.

Tilen, who proposes Go, is a beginner in Go and wants to learn it through this project.

Code is written and committed from developers' local machines; CI builds the code from what is committed.

## Decision
**Go is the default language for services.** A service owner may choose another language (for example TypeScript). That is allowed, provided the service keeps the shared contracts and operational rules: OpenAPI spec (A5), agreed event payloads (E2), stdout logging (R2), `/metrics` (R3) and health endpoints (R4).

Baseline for Go services:

- Tests: the standard `go test` with table-driven tests and subtests. No Ginkgo/Gomega.
- Lint: `golangci-lint` with the `errcheck` linter enabled in each Go service's CI pipeline. Go does not force callers to handle returned errors, so CI must.
- Images: built with a Dockerfile. ko is **not** used for now; add it only if we notice it is missing.

## Alternatives considered
- **Java (Spring Boot / KumuluzEE), suggested by the assistant.** Not chosen as default. Our reasons: higher memory use and slower startup, a framework needed for cloud-native features, and an image that must carry a JVM. Note: much of the startup and memory cost comes from the framework, not the JVM itself, and Java 21 virtual threads and GraalVM native images narrow the concurrency and startup gaps. The decision rests on team fit and operational simplicity, not on raw performance.
- **TypeScript / Node.js.** Not chosen as default. Our reason: heavy toolchain (npm, transpiling, bundling) and a Node runtime underneath. Still allowed per service.
- **Rust.** Considered. Go chosen because it is faster and more ergonomic to develop in.

## Consequences
- **Positive:**
  - Single static binary, small images, no runtime to package in the container.
  - First-class client libraries and examples for Kubernetes and NATS.
  - Errors are returned as values, so error paths are visible in the code.
- **Negative / accepted risks:**
  - Go does not force error handling; without the `errcheck` gate an error can be silently ignored.
  - Go has fewer batteries-included frameworks: we choose and wire libraries ourselves (router, DB driver, config, and `nats.go` with its `jetstream` package as the NATS client) instead of one framework doing it. Simple does not mean less code.
  - The proposer is a Go beginner, so the default language is also a learning cost. `errcheck` and the other `golangci-lint` checks help catch common mistakes early.
  - Allowing deviation means more than one build, test and lint setup, and reviewers can only review code in languages they know.
  - Only the Dockerfile route is known to the whole team; ko adds a second image-build path if introduced later.
- **Follow-ups:**
  - Decide which services (if any) deviate from Go, after service boundaries are defined.
  - Add `golangci-lint` with `errcheck` to the CI pipeline of the first Go service (tracked in `_ToDo`).
  - **Revisit trigger for ko:** the Dockerfile build becomes slow or repetitive in CI.

## Assumptions to validate
- The assistant and professor accept non-Java languages. The PDF has no language restriction; the assistant's and professor's position is as stated by Tilen and not yet confirmed in writing.
- Tilen is a beginner in Go and wants to learn it through the project. TODO (user): which other teammates already know Go, and how many hours the others need to get productive. Not yet checked.
