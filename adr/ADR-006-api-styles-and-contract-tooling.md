# ADR-006: REST for the GUI, one gRPC link and one GraphQL endpoint, OpenAPI as the contract

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-002, ADR-005; rules A1, A5, A6, C1, E2; project requirements #5 (API documentation), #15 (GUI over REST), #17 (GraphQL and gRPC)

## Context
- Requirement #15 (mandatory) says the GUI talks to the microservices over REST APIs.
- Requirement #5 (3 points) wants OpenAPI or Postman documentation per service.
- Requirement #17 (4 points) needs GraphQL for querying data in **one** microservice and gRPC for communication between **two** services. It does not require them everywhere.
- Services talk to each other mainly through events (ADR-001); synchronous calls are only for cases where the caller needs the result (rule C1).
- The team is mixed-language (ADR-005) and mixed-experience, so the API contract must be language-neutral and add as little extra tooling as possible.

## Decision
- **GUI to backend:** REST, documented in OpenAPI 3.1 with RFC 9457 errors (rules A5, A6).
- **Service to service:** events over NATS JetStream by default (ADR-001, ADR-002). Synchronous calls only when the caller needs the result (rule C1).
- **For requirement #17:** exactly one gRPC link between two services and one GraphQL endpoint in one service. The scope is deliberately limited. TODO (user): which services own them, decided after service boundaries are defined (tracked in `_ToDo`).
- **Contract tooling:** the OpenAPI file is the source of truth of each REST API. Code is generated from it: `oapi-codegen` for Go, `openapi-typescript` for TypeScript. Postman is only a client for calling and testing APIs (it can import the OpenAPI file); it is not the contract.

## Alternatives considered
- **gRPC for all synchronous service-to-service calls.** Rejected: adds protobuf tooling and a second contract style to every service in a mixed-skill team, and most inter-service traffic is events anyway. Kept for one pair.
- **GraphQL for the GUI.** Rejected: requirement #15 asks for REST from the GUI. GraphQL stays in one service.
- **Postman collections as the API documentation instead of OpenAPI.** Rejected: rule A5 asks for OpenAPI 3.1, and code generation needs the spec.
- **Skipping requirement #17.** Rejected: 4 points for limited scope.

## Consequences
- **Positive:**
  - One language-neutral contract shared by Go and TypeScript through code generation.
  - 4 points for requirement #17 at the cost of two small integrations.
  - Most services need REST plus events only.
- **Negative / accepted risks:**
  - Two extra technologies (protobuf tooling, a GraphQL server) for a few services.
  - Generated code must be regenerated whenever a spec changes.
  - The gRPC pair must be a genuine synchronous need under rule C1; forcing it would conflict with the events backbone.
- **Follow-ups:**
  - Decide which two services use gRPC and which one exposes GraphQL.
  - TODO (user): decide whether generated code is committed or generated in CI.

## Assumptions to validate
- TypeScript teammates can use gRPC and consume the generated clients without extra friction. Not yet checked.
