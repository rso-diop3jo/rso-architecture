# ADR-010: Event sourcing and CQRS in the `character` service

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-002, ADR-008; rules E2, E4, E5, I4, DB1, DB3, T1; project requirement #12 (event sourcing and CQRS)

## Context
- Requirement #12 (5 points, optional) asks for event sourcing to track data changes, CQRS with separate read and write models, and a demo of replaying events to rebuild state.
- ADR-001 makes events the backbone **between** services. Event sourcing is a different thing: it is how **one service stores its own data**.
- The `character` service owns XP, level, streak, trophies and equipment (see `koncept_in_meje_storitev.md`). XP is naturally a ledger: it only changes by awards, and we want to be able to answer "why is this character level 5?".
- The service is read-heavy (profile screen, GraphQL profile query, `GetPartySnapshot` for `story`, see ADR-006) and write-light (one award per completed quest).
- Event delivery is at-least-once (rule I4), so the same `QuestCompleted` can arrive twice.

## Decision
We will store the state of a character as an **append-only sequence of events** (the source of truth) in the `character` service only, and read it through a separate **read model**.

- **Write side:** table `character_events` (id, character_id, tenant_id, type, payload, version, time), append only. A command such as `AwardXp` checks the rules and appends events such as `XpAwarded` and `LevelReached`. `version` gives optimistic concurrency per character.
- **Idempotency:** the ID of the incoming `QuestCompleted` is stored with the resulting event, so a second delivery is detected and skipped (rule I4).
- **Read side:** table `character_view` (level, XP, streak, trophies, ...), updated by a projector from the events. The GraphQL profile query and `GetPartySnapshot` read from the view, never from the event table.
- **Replay:** a `rebuild` command empties the view and rebuilds it from the events. The result must be identical. This is the demo for requirement #12.
- **Internal and published events are different things.** Events in `character_events` are private to the service and may change. What other services see are the events published to NATS, whose payloads are agreed in the event catalogue (rule E2).
- `tenant_id` is stored on every event and every view row (rule T1).

## Alternatives considered
- **Plain current-state table (`xp = 340`).** Simplest, and what most services will do. Rejected for `character`: no history, no way to explain or rebuild a value, and no replay demo for #12.
- **Event sourcing in every service.** Rejected: more concepts and code for services that mostly store current state (for example `group`, `notification`), in a team where some members are still learning Go.
- **Event sourcing in `quest` instead.** Rejected: a quest has a short lifecycle (created, completed or expired), so replay shows little. XP is a growing ledger.
- **Audit-log table next to a current-state table.** Rejected: two sources of truth that can disagree, and it does not give "replay to rebuild state".

## Consequences
- **Positive:**
  - Full history of every XP change. A value can be explained, and a bug in the rules can be fixed by replaying the events.
  - Reads are cheap and can be shaped for the GUI and for the LLM context.
  - 5 points for requirement #12.
- **Negative / accepted risks:**
  - More code and concepts than a plain table (command handler, projector, rebuild), in a language the owner is still learning.
  - If the projector is asynchronous, the view can lag behind the events (eventual consistency, rule DB3).
  - Changing the shape of a stored event later needs versioning of stored events (rule E5 applies to the store too).
  - The dual-write problem (rule E4) remains: appending an event and publishing it to NATS must not lose one of them. With event sourcing the event store can act as the outbox, but the strategy is still open (candidate ADR-004).
- **Follow-ups:**
  - TODO (user): decide whether the projector runs in the same database transaction as the append (simple and consistent) or asynchronously. Recommendation for the first version: same transaction.
  - Agree the event catalogue (stored and published events with payloads) after the use cases (tracked in `_ToDo`).
  - Decide the event-store details: payload as JSON, versioning of event shapes.

## Assumptions to validate
- Any store with a unique constraint on (`character_id`, `version`) supports the optimistic concurrency. A relational database is the simplest fit. The database type per service is not decided yet (rule DB4).
- Replaying the events of one character is fast enough for the demo, because streams are short and no snapshots are needed at first. Check in the first spike.
