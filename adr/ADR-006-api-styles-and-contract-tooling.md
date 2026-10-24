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
- **For requirement #17:** exactly one gRPC link between two services and one GraphQL endpoint in one service. The scope is deliberately limited. The gRPC link is `story` → `character` (`GetPartySnapshot`). The GraphQL endpoint is in `character` (rich profile query). Services decided 2026-10-24; which team member implements each is tracked in `_ToDo`.
- **Why `story` → `character`:** at the end of the day `story` needs the level, equipment and recent actions of all party members to build the LLM prompt. That is one large typed response with a single call (like batch rule A2) and a deadline that fits the LLM window. Rule C1: `story` cannot continue without the answer. Rule E3: the event is the news, the details come from the API of the publisher.
- **Contract tooling:** the OpenAPI file is the source of truth of each REST API. Code is generated from it: `oapi-codegen` for Go, `openapi-typescript` for TypeScript. Postman is only a client for calling and testing APIs (it can import the OpenAPI file); it is not the contract.

## Alternatives considered
- **gRPC for all synchronous service-to-service calls.** Rejected: adds protobuf tooling and a second contract style to every service in a mixed-skill team, and most inter-service traffic is events anyway. Kept for one pair.
- **gRPC between `quest` and `group` (membership check).** Rejected: small payload, REST would work the same, and membership can be kept from events.
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
  - Assign team members to the gRPC link and the GraphQL endpoint (tracked in `_ToDo`).
  - TODO (user): decide whether generated code is committed or generated in CI.

## Assumptions to validate
- TypeScript teammates can use gRPC and consume the generated clients without extra friction. Not yet checked.
