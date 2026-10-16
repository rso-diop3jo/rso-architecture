# RSO project: architecture workspace

Design workspace for the RSO (Računalniške storitve v oblaku) team project: 1 GUI + 6 microservices on Kubernetes, with events over NATS JetStream.

It holds the architecture decisions (ADRs), the rules the design is checked against, and an AI coach, the `architect` subagent, that helps design the system.

## Layout

| Path | What it is |
|------|------------|
| `.claude/agents/architect.md` | The `architect` subagent definition |
| `architecture/rules.md` | Rules the architect enforces (lecture notes + corrections) |
| `adr/` | Architecture decision records (`ADR-001` ...) and `TEMPLATE.md` |
| `[SLO] Navodila za projekt in kriterij ocenjevanja.pdf` | Project requirements and grading (the architect reads it when scope or points matter) |

## Using the architect

Requires [Claude Code](https://claude.com/claude-code).

1. Open a terminal in the repository root.
2. Start Claude Code: `claude`
3. Call the agent with `@architect` followed by your request.

The architect is a **coach, not a designer**. It asks one question at a time, challenges your boundaries and contracts against `architecture/rules.md`, and points you to what to research. It does not give design options or recommendations: you do the designing. It does correct false factual claims, with a source.

### What you can ask

- **Design:** `@architect help me split the domain into 6 services`
- **Review:** `@architect review adr/ADR-001`
- **Write down a decision:** `@architect I decided X because Y, write it as an ADR`
- **Schemas:** `@architect let's define the payload of the answer-submitted event`

### Use case: review an ADR

```
@architect review adr/ADR-001
```

The architect reads `architecture/rules.md` and the existing ADRs, then reviews `adr/ADR-001-events-as-backbone.md`. It returns findings ranked Critical > High > Medium > Low. Each finding names where the problem is, which rule it breaks, why it matters, and what to rethink or look up. It does not hand you the finished fix.

## Working with ADRs

1. Reach a decision by discussing it with the architect.
2. Ask it to write the ADR. It uses `adr/TEMPLATE.md` and the next free number, with status `Proposed`.
3. Fill any `TODO (user)` line it leaves: alternatives, consequences and risks must come from you.
4. Change the status to `Accepted` when the team agrees.
5. To change an Accepted decision, write a new ADR that supersedes it. Do not edit the old decision.

## Editing the rules

`architecture/rules.md` is the single source of rules. Change it when a rule changes: the architect reads it at the start of every session. Rule IDs (D1, C2, I2 ...) are cited in review findings.
