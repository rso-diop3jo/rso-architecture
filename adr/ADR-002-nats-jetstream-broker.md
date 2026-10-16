# ADR-002: NATS with JetStream as the message broker

- **Status:** Proposed
- **Date:** 2026-10-16
- **Deciders:** RSO project team
- **Related:** ADR-001, ADR-003; rules C2, I4, E2; project requirement #18 (messaging system)

## Context
ADR-001 needs a broker that carries business events between services. Requirements:

- Easy for services to subscribe to the channels they care about.
- Messages must not be lost if the broker restarts. Plain NATS (core) delivers messages only to subscribers connected at that moment, so persistence is needed.
- We do not want to invent our own delivery guarantees or retry logic.
- Small team, small budget (Azure student credit), and the course lecturer prefers NATS over Kafka.

The two candidates were Apache Kafka and NATS with JetStream.

## Decision
We use **NATS with JetStream enabled**. Services publish and subscribe to business-event subjects. JetStream is built into the NATS server, so enabling it needs no extra service.

JetStream provides persistence (messages written to disk) and **at-least-once delivery**, so we rely on it instead of building our own fault tolerance for message sending.

## Alternatives considered
- **Apache Kafka.** Mature, widely used, brokers can be clustered, and it has a large ecosystem. Rejected for this project because it is heavier to operate than we need for 6 services and a student budget, and its defaults favour throughput over latency (batching; tunable with settings such as `linger.ms`). Kafka 4.0 no longer uses ZooKeeper (KRaft mode), so ZooKeeper is not a reason to reject Kafka.
- **Plain NATS without JetStream.** Rejected: no persistence, so events are lost if a consumer is down or the broker restarts.

## Consequences
- **Positive:**
  - Simple subscription model: services subscribe to subjects.
  - Persistence and at-least-once delivery come from JetStream, not from our code.
  - Lightweight to run on Kubernetes.
  - Streaming-oriented, low latency.
- **Negative / accepted risks:**
  - **At-least-once means duplicates are possible.** JetStream does not guarantee a message arrives exactly once. Every consumer must be idempotent (rule I4). Publisher-side duplicates can be filtered with the `Nats-Msg-Id` header inside the stream's duplicate window, but that does not remove duplicates on the consumer side.
  - Smaller ecosystem and fewer engineers with experience than Kafka.
  - Our deployment topology (single node) adds a single point of failure: see ADR-003.
- **Follow-ups:**
  - Define subject naming and the event envelope (see ADR-001).
  - Decide how tenants are separated in the broker (subject, header, NATS accounts).
  - Define consumer durability, acknowledgement and retry settings.

## Assumptions to validate
- "NATS is easier to set up than Kafka" is **not yet tested**. Validate with a short spike: deploy NATS + JetStream on the cluster and publish/consume one event between two services. Estimate: about 1-2 hours.
