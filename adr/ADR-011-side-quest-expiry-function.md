# ADR-011: Side quests expire through a delayed queue message that triggers an Azure Function in TypeScript

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-005, ADR-008, ADR-012, candidate ADR-004; rules A1, A3, A4, C1, D4, E4, I4, T1, T4, R2, R3, P1, P5, DB2; project requirement #16 (serverless function); UC-03

## Context
- Requirement #16 (optional, 5 points): a serverless function (Azure Functions recommended, AWS Lambda or Google Cloud Functions also allowed) that is triggered by events (for example an HTTP request), with its operation observed and its capacity adapting to the load.
- The team chose the 24-hour expiry of side quests as the serverless function (`koncept_in_meje_storitev.md`).
- UC-03: a side quest is valid for 24 hours. After that nobody can complete it (7c), and `SideQuestExpired` is published even when nobody completed it (flow E1). The app reacts by no longer offering the side quest.
- `quest` owns side quests: their state, XP, completions and expiry.
- First idea: a function listens on NATS for `SideQuestCreated`, keeps a 24-hour timer in memory and then publishes `SideQuestExpired`. Two facts rule it out (checked 2026-10-24 in the Azure Functions documentation):
  - Azure Functions has no NATS trigger. The supported triggers include HTTP, Timer, Queue Storage, Blob Storage, Service Bus, Event Grid, Event Hubs and Azure Cosmos DB.
  - A function execution cannot be relied on to run for 24 hours. Azure can stop or replace an instance at any time, and when the Flex Consumption plan scales in, a running execution gets at most 60 minutes of grace. A timer held in memory is then lost, and the waiting time is billed as execution time.
- The function is written in TypeScript (decided 2026-10-24). Native Go support in Azure Functions is in public preview (Flex Consumption plan only) and has no Queue Storage trigger. Durable Functions does not support Go.

## Decision
**`quest` schedules the expiry as a delayed message in an Azure Storage Queue. When the message becomes visible after 24 hours, it triggers an Azure Function written in TypeScript, which asks `quest` to expire the side quest. Only `quest` changes the side quest and publishes `SideQuestExpired`.**

Flow:

1. `quest` saves the side quest with `expires_at` = creation time + 24 hours and publishes `SideQuestCreated` on NATS (UC-03, step 4), as before.
2. In the same operation, `quest` puts the message `{ "tenantId", "sideQuestId", "expiresAt" }` into the Azure Storage Queue `side-quest-expiry`, with a visibility timeout of `expires_at` minus now (24 hours when sent right away). The message stays invisible until then. The maximum visibility timeout is 7 days.
3. When the message becomes visible, the Queue Storage trigger starts the function in `campfire-side-quest-expiry`. One message means one short execution.
4. The function calls `POST /v1/side-quests/{id}/expire` on the internal API of `quest` (rules A1, A3, A4), with the `tenantId` as tenant context (T1), a secret that proves the caller is the function, and a timeout (C1).
5. `quest` checks that the side quest belongs to that tenant (T4) and that `expires_at` has passed, marks the side quest expired and publishes `SideQuestExpired` on NATS. If the side quest is already expired, `quest` answers with success and changes nothing (idempotent, I4). If the call comes too early, `quest` answers `409 Conflict` and the message is retried.
6. `notification` forwards the expiry to the group's GUIs (ADR-012), which stop offering the side quest (UC-03).

Key details:

- Hosting: Azure Functions on the **Flex Consumption** plan, which Microsoft recommends for new serverless apps. Node.js with TypeScript, v4 programming model (`app.storageQueue(...)`).
- Own repository and pipeline: `campfire-side-quest-expiry` (rules D4, P1).
- The queue lives in an Azure Storage account. A Function App needs a Storage account anyway, so we use that account or a separate one for application data.
- Encoding: by default the Functions runtime expects queue messages encoded as Base64 (`messageEncoding` in `host.json`). Either `quest` Base64-encodes the JSON, or the function app sets `messageEncoding` to `none`. Pick one and write it in both READMEs.
- Retries: a failed execution is retried up to 5 times (`maxDequeueCount`), then the message moves to the queue `side-quest-expiry-poison`. `visibilityTimeout` in `host.json` sets the pause between retries.
- Precision: when the queue is idle, the trigger still polls it at least once a minute (`maxPollingInterval`), so a side quest expires at most about a minute after its 24 hours.
- `quest` does not depend on the function for correctness. On every completion it compares the time with `expires_at` and refuses late completions (UC-03, 7c). If it notices the expiry first, it publishes `SideQuestExpired` itself (UC-03).
- The function keeps no state and has no domain logic. It is a wake-up call: the rules stay in the `quest` use case that expires a side quest (ADR-008).

Sketch (not final code):

```ts
import { app, InvocationContext } from "@azure/functions";

type ExpiryMessage = { tenantId: string; sideQuestId: string; expiresAt: string };

app.storageQueue("expireSideQuest", {
  queueName: "side-quest-expiry",
  connection: "SIDE_QUEST_QUEUE",
  handler: async (item: unknown, context: InvocationContext): Promise<void> => {
    const msg = item as ExpiryMessage;
    const res = await fetch(`${process.env.QUEST_URL}/v1/side-quests/${msg.sideQuestId}/expire`, {
      method: "POST",
      headers: {
        "X-Tenant-Id": msg.tenantId, // placeholder, see follow-ups
        "X-Internal-Key": process.env.QUEST_INTERNAL_KEY ?? "", // placeholder, see follow-ups
      },
      signal: AbortSignal.timeout(5_000),
    });
    if (!res.ok) {
      // Throwing makes the runtime retry; after 5 attempts the message goes to side-quest-expiry-poison.
      throw new Error(`quest answered ${res.status} for side quest ${msg.sideQuestId}`);
    }
    context.log(`side quest ${msg.sideQuestId} expired (tenant ${msg.tenantId})`);
  },
});
```

```json
{
  "version": "2.0",
  "extensions": {
    "queues": {
      "maxDequeueCount": 5,
      "visibilityTimeout": "00:00:30"
    }
  }
}
```

## Alternatives considered
- **Function subscribed to NATS, with a 24-hour timer in memory** (the first idea). Rejected: Azure Functions has no NATS trigger, and a timer in memory does not survive an instance being stopped or scaled in (60 minutes of grace on Flex Consumption). We would also pay for 24 hours of waiting per side quest.
- **Durable Functions** (an orchestration started over HTTP, with a 24-hour durable timer). Works and survives restarts, but adds concepts we do not otherwise need (orchestrator replay and its determinism rules, a task hub in storage) and does not support Go. Kept as the fallback if the queue approach fails.
- **Timer trigger every few minutes that expires all due side quests** (a sweeper). The simplest to build, but it is a cron job: it runs even when nothing expires, and a timer function runs as a single instance, so it shows nothing about scaling with load (#16).
- **Azure Service Bus scheduled message instead of a Storage Queue.** Built for "deliver at time T", but needs another Azure resource (a Service Bus namespace), while the Function App already needs a Storage account. It would be the choice for a function in Go: the native Go preview supports Service Bus, not Queue Storage.
- **Expiry inside `quest` only, without a function** (a scheduler in the service, or noticing the expiry on the next completion). Rejected: loses requirement #16 (5 points), and noticing the expiry only on the next completion publishes `SideQuestExpired` late, or never when nobody tries to complete the side quest.

## Consequences
- **Positive:**
  - Durable storage does the waiting, not a running process. Nothing is lost when an instance stops, and we pay for one short execution per side quest.
  - One owner: only `quest` changes a side quest and publishes events about it. The function has no domain logic and no database (DB2).
  - Correctness does not depend on the function. A late or failed function only delays the event; nobody can complete an expired side quest.
  - The expire endpoint is idempotent, so retries and duplicate messages are harmless (I4).
  - Covers #16: an event (a queue message) triggers the function, Flex Consumption adds instances when many side quests expire at once, and executions, durations and failures are visible in Application Insights.
- **Negative / accepted risks:**
  - A second queue technology next to NATS. It is used only as the function's timer, and the architecture diagram must show it that way.
  - `quest` writes to its database and to the queue: a dual write (E4). If the queue write fails, the side quest is still refused after 24 hours, but `SideQuestExpired` comes only when someone tries to complete it.
  - The function runs outside the AKS cluster, so the expire endpoint of `quest` must be reachable from outside (through the ingress) and must reject callers without the secret. The alternative is VNet integration of the Function App: more Azure networking work.
  - The function logs to Application Insights, not to our central logging (R2), unless we forward the logs. Two places to look.
  - Precision is about one minute, and a visibility timeout cannot exceed 7 days. A side quest longer than 7 days would need another approach.
  - The function is TypeScript while the services default to Go (ADR-005 allows per-service deviation).
- **Follow-ups:**
  - Add the payloads of `SideQuestCreated` and `SideQuestExpired` to the event catalogue (E2, `_ToDo`).
  - When ADR-004 (dual write) is decided, the queue message goes through the same mechanism as the NATS events (for example, the outbox relay sends both).
  - Decide how the tenant context and the caller's identity travel on internal calls (T2). The header names in the sketch are placeholders.
  - Decide where the secrets live (P5): the Storage connection for `quest`, and the `quest` URL and secret for the function.
  - Alert when messages land in `side-quest-expiry-poison` (R2, R3).
  - TODO (user): who owns the function and its repository.

## Assumptions to validate
- A queue-triggered Azure Function that calls our service counts as "triggered by events" with "capacity adapting to the load" for #16. Ask at the next vaje.
- The Flex Consumption plan is available in the Azure region we use (the vaje found that only a few regions work with the student subscription). Check before creating the Function App.
- The Queue Storage trigger works on Flex Consumption with Node.js and the v4 programming model as documented (checked 2026-10-24). Verify with a spike: enqueue a message with a 2-minute visibility timeout and watch the function run.
- Azure for Students credits cover the Function App, the Storage account and Application Insights at our volume. Check the cost after the first week.
