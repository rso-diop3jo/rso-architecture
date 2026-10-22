# ADR-009: React Native GUI in its own repository, talking to the backend over REST

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-005, ADR-006; rule D4; project requirements #15 (GUI), #8 (cloud deployment)

## Context
- Requirement #15 (mandatory, 4 points): a GUI built with a framework (React, Angular, Vue.js) **or a mobile application**, with at least 2 subpages, communicating with the microservices over REST APIs (checked in the project PDF, 2026-10-22).
- The GUI is counted separately from the minimum of two microservices per member (requirement #2, rule D4).
- Some teammates write TypeScript (ADR-005), and the REST contracts are OpenAPI files with generated TypeScript clients (ADR-006).
- Requirement #8 (mandatory, 5 points): the Kubernetes cluster and the application must be publicly accessible.
- Learners sign in with Google (UC-01).

## Decision
The GUI is built with **React Native** and lives in **its own repository**, separate from the services. It calls the backend through the services' REST APIs (requirement #15, ADR-006).

## Alternatives considered
TODO (user): what else did you consider (for example another framework from requirement #15), and why did it lose?

## Consequences
- **Positive:**
  - Teammates who write TypeScript can work on the GUI, and a typed client can be generated from the OpenAPI files (ADR-006).
  - A GUI change is built and deployed without touching the backend repositories.
- **Negative / accepted risks:**
  - TODO (user): which risks do you accept with React Native?
- **Follow-ups:**
  - Define the first two subpages (the requirement minimum is 2).
  - TODO (user): how does the GUI learn the result of a long operation without blocking? The vaje suggested the UI subscribes to events, while #15 says REST (see rule C3: `202 Accepted` + a status resource).

## Assumptions to validate
- How the GUI is "publicly accessible" and demonstrated for requirement #8 when it is a React Native app. Not yet checked; ask at the next vaje (tracked in `_ToDo`).
- Google sign-in (UC-01) works the same in a React Native app as on the web. Not yet checked.
