# ADR-008: Clean architecture inside every service

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-005; rules S1-S5, D1, DB2; project requirement #2 (clean architecture concepts are in its minimum requirements)

## Context
- Requirement #2 (mandatory, 4 points) lists "concepts of clean architecture" among its minimum requirements.
- Each service is owned by one person, lives in its own repository (D4) and may use a different language (ADR-005). The course assistant said (vaje 2026-10-22) that different technologies per service are fine as long as everyone knows where things are in each other's code.
- A service talks to the outside in two directions: it receives requests (its API) and it calls out (other services, storage). Code for that mixed into the business logic makes the logic hard to test and the service hard to run in different environments (in-memory for tests and dev pipelines, a real database in staging and production).

## Decision
Every service is structured as a clean-architecture "onion", as four folders in its repository:

- `domain` (innermost ring): the elementary classes and functions, e.g. the classes that define the data the service keeps.
- `usecases`: everything the service offers and does.
- `web` (outer ring): the service's API, incoming traffic.
- `adapters` (outer ring): outgoing traffic, e.g. calls to other services (storage too, see rules S4-S5).

Dependencies point inward only. Services talk to each other only through public APIs or events, never by importing each other's code. Full wording: rules S1-S5 in `architecture/rules.md`.

## Alternatives considered
TODO (user): what else did you consider, and why did it lose?

## Consequences
- **Positive:**
  - Business logic does not depend on the API or on storage, so it can be tested without them.
  - The same layout in every service: a teammate can find where things are even in a service written in another language.
  - Storage can be swapped by configuration (in-memory or real database), which fits the per-environment Helm values (rule P4).
  - Covers the clean architecture concepts asked for by requirement #2.
- **Negative / accepted risks:**
  - More folders and interfaces than a small service strictly needs.
  - The wrong import direction (for example the domain importing an adapter) is not stopped by default; it needs review or an automatic check. Whether tooling enforces it differs per language: not checked.
- **Follow-ups:**
  - TODO (user): is the import direction checked automatically in CI, or by review only? (tracked in `_ToDo`)
  - TODO (user): does every service get an in-memory storage adapter for tests and dev pipelines? (tracked in `_ToDo`)
  - TODO (user): describe the first defined service ring by ring: what lives in the domain, what in the use cases.

## Assumptions to validate
- The four-folder layout is not too heavy for a service with little logic. Not yet checked: look at it when the first service is built.
- TypeScript teammates can use the same layout in their services. Not yet checked.
