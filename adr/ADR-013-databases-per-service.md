# ADR-013: PostgreSQL for `identity`, `group`, `quest` and `character`; MongoDB for `story` and `notification`

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** Tilen (proposed); to be confirmed by the RSO project team
- **Related:** ADR-001, ADR-005, ADR-008, ADR-010, candidate ADR-004; rules DB1, DB2, DB4, E4, E5, T1, T3, T4, S5, P4, P5, R2; project requirements #3 (database per service), #10 (multi-tenancy), #12 (event sourcing and CQRS), #19 (centralized logging)

## Context
- Requirement #3 (mandatory, 2 points): a relational (PostgreSQL, MySQL) or NoSQL (MongoDB) database. Every service with persistent data has its own database. Services never access another service's database directly, only through APIs or messages. The databases hold some tables and data.
- Rules DB1 and DB2: one database per service, never a shared database, and other services' data only through APIs or events. Rule DB4 (lecture): choose the database that fits each service (for example a search, document or vector store for a catalogue, and a relational database for purchases, where integrity matters).
- Requirement #10 (optional, 5 points): multi-tenancy at the level of the services, Kubernetes and the database. The tenant is a group of friends, and `tenant_id` travels through the whole chain (T1). The isolation model (T3) is not decided.
- ADR-010: `character` is event-sourced, with the append-only table `character_events` (optimistic concurrency on `character_id` and `version`) and the read model `character_view`. ADR-010 left the database type open.
- A service writes its data and publishes events in one operation (E4). The strategy is open (candidate ADR-004).
- Storage sits behind an adapter (ADR-008, rule S5), so the database choice stays inside one ring of one service.
- Every technology we add must be run, paid for (Azure for Students credits) and learned by a team of three.
- The data each service keeps:

| Service | Data |
|---|---|
| `identity` | traveler profiles linked to a Google account (Google `sub`, e-mail, name) |
| `group` | groups (`tenant_id`, name), members, invitations, the leaderboard, the daily overview (who chose and completed what) |
| `quest` | default quests, travelers' recurring activities with their repeat and rated XP, daily choices, completions, side quests with their XP, completions and expiry |
| `character` | the events of every character (`character_events`) and the read model (`character_view`): XP, level, streak, trophies, equipment |
| `story` | campaigns (goal, theme, progress), story chapters written by the LLM, route options and choices |
| `notification` | notifications sent to travelers, kept for a limited time, and reminder settings |

## Decision
**`identity`, `group`, `quest` and `character` use PostgreSQL. `story` and `notification` use MongoDB. Every service has its own database and its own database user, and no service can read another service's database.**

| Service | Database | Why |
|---|---|---|
| `identity` | PostgreSQL | The Google `sub` and the e-mail must be unique: unique constraints. |
| `group` | PostgreSQL | Groups, members and invitations are relations with constraints (one membership per traveler and group). The leaderboard is an `ORDER BY` over a handful of members. |
| `quest` | PostgreSQL | "One choice per day" and "one completion of a side quest per friend" are unique constraints, checked in a transaction. Counting repeats in the current day, week or month is a SQL query. |
| `character` | PostgreSQL | `character_events` with a unique constraint on (`character_id`, `version`) gives the optimistic concurrency, and `character_view` can be updated in the same transaction as the append (ADR-010). |
| `story` | MongoDB | Chapters written by the LLM are nested documents that are read whole, and their shape changes when the prompts change. Adding XP to a campaign's progress is an atomic update of one document. |
| `notification` | MongoDB | A TTL index deletes old notifications automatically, and each notification type has different content. |

Key details:

- **Why PostgreSQL for most services:** besides the reasons in the table, row-level security (RLS) can enforce the `tenant_id` filter inside the database, which shows multi-tenancy at the database level (#10, T4). A transactional outbox (E4, candidate ADR-004) is then a table written in the same transaction as the data. One technology for four services means one way to run, back up and migrate them.
- **Why MongoDB for only two services:** it is used only where the data has the shape of documents. Two technologies show that the database was chosen per service (DB4) and stay manageable for a team of three.
- **One PostgreSQL server, four databases:** the four PostgreSQL databases may share one server, each with its own database and its own user that can reach only that database. The data stays private to each service (DB1), and we pay for one server instead of four. The same applies to the two MongoDB databases. The pattern "database per service" allows a shared server as long as each service's data stays private ([microservices.io](https://microservices.io/patterns/data/database-per-service.html)).

## Alternatives considered
- **PostgreSQL for all six services.** One technology to run and learn, and JSONB could hold the chapters. Rejected: the chapters and the notifications are documents with a changing shape and a limited lifetime, and a single technology would not show the per-service choice of DB4.
- **MongoDB for all six services.** Flexible documents everywhere. Rejected: the data of `identity`, `group`, `quest` and `character` is relational (memberships, invitations, completions that refer to quests and travelers), and its rules span several records in one transaction (for example a completion, the daily limit and the outbox), which in MongoDB needs multi-document transactions on a replica set. MongoDB also has no equivalent of row-level security, so the tenant filter would rely on code in every service.
- **Elasticsearch (a search engine).** Rejected for now: no service has search as its main job. As the main store it has no transactions across documents, only the document `_id` is unique, a new document becomes searchable only after a refresh (within about a second), and it needs a lot of memory (JVM). For centralized logging (#19) the lecture names Grafana Loki (R2), which indexes only labels and is cheaper. Revisit for the shared town after the MVP: a public, read-only projection of groups and trophies that people search and browse is the catalogue case of DB4.
- **A dedicated event store for `character`** (KurrentDB, formerly EventStoreDB). Rejected: one more technology to run and learn. A PostgreSQL table with a unique constraint on (`character_id`, `version`) gives the optimistic concurrency, and the replay demo for #12 works with it.
- **Redis for the leaderboard** (sorted sets). Rejected: a group has a handful of members, so an `ORDER BY` is enough, and Redis would be one more technology.

## Consequences
- **Positive:**
  - Each database fits its data: the database enforces the integrity rules where they matter, and documents stay documents.
  - Covers #3, and RLS gives a way to show multi-tenancy at the database level (#10).
  - The event store and the read model of `character` (ADR-010) live in one database, so the projector can run in the same transaction as the append (#12).
  - Only two technologies: the team learns PostgreSQL well, and MongoDB only for the two document services.
- **Negative / accepted risks:**
  - Two database technologies to run, back up and connect to (connection settings, secrets and Helm values per environment: P4, P5).
  - RLS does not apply to superusers, to roles with `BYPASSRLS` and, by default, to the owner of the table. A service must connect with a role that does not own its tables, or the tables use `FORCE ROW LEVEL SECURITY`.
  - MongoDB has no RLS. In `story` and `notification` the storage adapter enforces the tenant filter, and every query must include `tenant_id` (T4).
  - MongoDB multi-document transactions need a replica set. An outbox in `story` either lives in the same document as the change or uses a transaction on a replica set.
  - A TTL index deletes expired documents in a background task that runs every 60 seconds, so deletion is not immediate. Fine for old notifications.
  - A shared server is a shared point of failure, and one busy service slows the others. Accepted to save cost.
- **Follow-ups:**
  - Decide where the databases run and who pays for them (open point in `architecture/rules.md`): managed Azure services or inside the cluster.
  - Decide the tenant isolation model (T3). PostgreSQL supports all three options: a shared database with RLS, a schema per tenant, a database per tenant.
  - Choose a schema migration tool for the PostgreSQL services, and version the MongoDB documents (a `schemaVersion` field; expand, migrate, contract: E5).
  - Decide how long notifications are kept (the TTL).
  - Update ADR-010, which still says the database type is not decided.

## Assumptions to validate
- PostgreSQL and MongoDB have mature drivers for Go and for TypeScript (ADR-005). Check in the first spike in each language.
- One shared PostgreSQL server with separate databases and users is accepted for requirement #3. Ask at the next vaje.
- RLS with a tenant setting per request works with the connection pool of the chosen driver: the tenant must be set at the start of every transaction (for example `set_config('app.tenant_id', ..., true)`). Check in a spike.
