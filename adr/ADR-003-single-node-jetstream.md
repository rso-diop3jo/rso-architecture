# ADR-003: Run JetStream as a single node (accepted risk)

- **Status:** Proposed
- **Date:** 2026-10-16
- **Deciders:** RSO project team
- **Related:** ADR-002; rules C4, I3, I4; project requirements #13 (fault isolation and tolerance), #18

## Context
JetStream can run as a cluster with replicated streams (replication factor 3, Raft-based), which keeps messages available when one server fails. A cluster needs at least 3 servers.

Our environment is a course project on one Azure Kubernetes cluster paid from limited student credit (resource groups and the cluster cost money continuously). We want a working, demonstrable event backbone, not production-grade availability.

Running a single node is a **deployment choice**, not a limitation of NATS.

## Decision
We run **one NATS server with JetStream enabled** and streams with **replication factor 1 (R1)**. Stream data is stored on a **persistent volume**, so messages survive a pod restart.

This is a conscious decision made to save cost and complexity, and we accept the risks below.

## Alternatives considered
- **3-node NATS cluster, streams with R3.** Survives the loss of one node and rolling upgrades without downtime. Rejected for now: three servers use more of the limited cluster resources and credit, and add operational effort we do not need for the project.
- **Multi-broker Kafka.** Same availability benefit as a cluster, with more operational weight. See ADR-002.

## Consequences
- **Positive:**
  - Lowest cost and simplest deployment.
  - Easier to debug during development.
- **Negative / accepted risks:**
  - **Single point of failure.** While the NATS pod is down or restarting, no events are published or delivered between services.
  - **Data loss if the persistent volume is lost.** Persistence protects against a pod or broker restart, not against loss of the disk.
  - Upgrades of the broker cause downtime.
  - It weakens the fault-tolerance story for requirement #13 unless services are built to tolerate a broker outage.
- **Mitigations / design rules that follow:**
  - Publishers use timeout and retry with backoff (rule I3) and must not lose an event when the broker is briefly unavailable. How is an open decision (candidate ADR-004, transactional outbox).
  - Consumers are idempotent (rule I4).
  - Services expose readiness checks that report broker connectivity, and degrade gracefully when it is down.
  - Monitor the NATS pod and its volume with the metrics stack.
- **Follow-ups:**
  - Decide the publisher-side strategy for a broker outage (ADR-004).
  - **Revisit trigger:** if the budget allows before the final presentation, or if we want to demonstrate fault tolerance for requirement #13, move to a 3-node cluster with R3.

## Assumptions to validate
- Moving from one node to a cluster should need deployment and stream-configuration changes but **no change to service code**, because services use only the NATS client API. Not yet verified; check during the spike in ADR-002.
