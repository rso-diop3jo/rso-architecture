# ADR-012: The GUI gets live updates over one WebSocket to `notification`, then reads the data over REST or GraphQL

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-006, ADR-009, ADR-011; rules C3, C5, E2, I1, I4, S3, T2, T4, R3; project requirement #15 (GUI); UC-03, UC-04

## Context
- Answers the open follow-up in ADR-009 and the last open point in `architecture/rules.md`: how does the GUI learn the result of a long operation without blocking?
- UC-04: at step 5 the app tells the traveler right away that the quest is done and the reward is on its way, and the traveler does not wait. At step 10 the traveler gets a live message on a level up or a new trophy.
- UC-03: friends are told about a new side quest, the creator is told when a friend joins, and the app stops offering a side quest once it expires.
- Requirement #15 (mandatory): the GUI talks to the microservices over REST APIs. ADR-006: REST with OpenAPI for the GUI, and GraphQL on `character`.
- Rule C3: a long operation answers `202 Accepted` with a status resource. The vaje suggested that the UI subscribes to events.
- Services integrate through NATS events (ADR-001). Event payloads are the coupling between our services (E2).
- React Native has WebSocket built in. It has no built-in EventSource for server-sent events (checked 2026-10-24).
- `notification` owns reminders and notifications (`koncept_in_meje_storitev.md`).

## Decision
**The GUI keeps one WebSocket connection to `notification`. A message only says what changed. The GUI then reads the new data over REST, or over GraphQL on `character`.**

Key details:

1. Commands and reads stay REST (OpenAPI, ADR-006). A long command answers `202 Accepted` (C3).
2. After sign-in, the GUI opens one secure WebSocket connection (`wss`) to `notification`, authenticated with the traveler's token. `notification` refuses connections without a valid token.
3. `notification` subscribes to the NATS events the GUI cares about (for example `XpAwarded`, `LevelUp`, `TrophyUnlocked`, `SideQuestCreated`, `SideQuestCompleted`, `SideQuestExpired`, `CharacterWounded`). From the event's `tenant_id` and travelers it decides who must be told (T4), and sends each of them a small message with the type and the IDs, never the event payload. Example: `{ "type": "level_up", "characterId": "..." }`.
4. When a message arrives, the GUI shows a notice where needed and reads the affected data again over REST or GraphQL. REST and GraphQL stay the source of truth.
5. After a lost connection, the GUI reconnects with backoff and reads the screens it shows again. Messages sent while it was offline are not replayed.

## Alternatives considered
- **The GUI subscribes to NATS directly** (as suggested at the vaje; NATS can accept WebSocket clients). Rejected: internal event payloads would become a public API that the app depends on (E2, C5), every phone would need broker credentials limited to its own tenant and traveler (T2, T4), and the broker would be exposed to the internet.
- **One WebSocket to each service.** Rejected: six connections and six places that authenticate and track clients, and every service would have to know about the GUI. `notification` already owns who is told what.
- **Polling only** (`202 Accepted` and a status resource, C3). Rejected as the main mechanism: the GUI either learns late or asks often for nothing. The status resource stays for checking one specific command.
- **Server-sent events (SSE).** Rejected: not built into React Native (needs a library), while WebSocket is.
- **Mobile push notifications** (APNs or FCM, for example through Expo). Not a replacement: they reach the phone when the app is closed, but they are not meant to keep an open screen up to date. A possible later addition for side-quest invitations.

## Consequences
- **Positive:**
  - One real-time channel with one owner. The other services do not know that clients exist (S3).
  - Messages carry no domain data, so event payloads stay internal. The GUI depends only on the REST and GraphQL contracts and a short list of message types.
  - The GUI never blocks (UC-04, step 5), and rewards and new side quests still show up at once.
  - Duplicate messages are harmless: the GUI only reads again (I1, I4).
- **Negative / accepted risks:**
  - `notification` keeps open connections, so its pods are not stateless. A deploy or a restart drops the connections, and the GUI must reconnect.
  - With more than one replica of `notification`, every replica must receive every relevant event (no shared consumer that splits the messages between replicas) and send only to the travelers connected to it.
  - The ingress must allow WebSocket upgrades and long-lived connections (timeouts).
  - One extra request per message (notify, then read). Fine at our volume.
- **Follow-ups:**
  - List the message types and their fields in the README of `campfire-notification`. That list is the contract between the GUI and `notification`.
  - Decide how the token is passed when the connection opens (T2).
  - Metrics: open connections and messages sent per type (R3).
  - Close the follow-up in ADR-009 and the open point in `architecture/rules.md` with a reference to this ADR.

## Assumptions to validate
- A WebSocket next to REST is fine for requirement #15, since REST stays the only way to change or read data. Ask at the next vaje.
- The React Native WebSocket can send the token the way we choose (a header or the first message). Check in a spike.
