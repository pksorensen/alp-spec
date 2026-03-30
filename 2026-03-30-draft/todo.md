# Assembly Line Spec — TODO & Progress

## Document Status

| Document | Status | Notes |
|---|---|---|
| `README.md` | DRAFT | ALP name added |
| `spec/01-overview.md` | DRAFT | Full content written |
| `spec/02-concepts.md` | DRAFT | Full glossary written |
| `spec/03-assembly-lines.md` | DRAFT | Full content written |
| `spec/04-stations.md` | DRAFT | Real schema from implementation |
| `spec/05-tasks.md` | DRAFT | Full lifecycle + real fields |
| `spec/06-transition-rules.md` | DRAFT | v1 + agentic conditions future section |
| `spec/07-server.md` | DRAFT | Real API surface, auth WIP noted |
| `spec/08-client.md` | DRAFT | Completion schema, polling, auth |
| `spec/09-operator.md` | DRAFT | Completion signaling, vibecast pattern |
| `spec/10-agent.md` | DRAFT | Assembly Line Repository, exit codes |
| `diagrams/architecture.drawio` | DONE | 4-role system architecture diagram |
| `diagrams/task-lifecycle.drawio` | DONE | State machine with all transitions |
| `diagrams/client-server-flow.drawio` | DONE | 9-step sequence diagram with phase bands |
| `examples/vibecheck.md` | DONE | Full 8-station pipeline, all prompts complete |
| `examples/hello-world.md` | DONE | Minimal 2-station example with curl + step table |

---

## Milestones

- [x] **M1 — Skeleton complete**: All documents have intent + section headings + embedded questions
- [x] **M2 — Core concepts written**: specs 01–06 have complete content
- [x] **M3 — Protocol specs written**: specs 07–10 have complete content (implementable)
- [x] **M4 — Diagrams drawn**: all three drawio files have proper diagrams
- [x] **M5 — Examples complete**: vibecheck + hello-world are fully worked examples
- [x] **M6 — Publishable**: `/spec/` route live — sidebar nav, markdown rendering, syntax highlighting

---

## Open Questions

### Remaining

*(no open questions)*

---

## Resolved Questions

| Q | Question | Decision | Date |
|---|---|---|---|
| Q1  | Formal protocol name? | **Assembly Line Protocol (ALP)** | 2026-03-30 |
| Q13 | Standardise MCP completion tool? | **Yes — `complete_station` tool; Operator MUST expose MCP server** | 2026-03-30 |
| Q2 | Name for the Operator role? | **Operator** (Station Operator in long form) | 2026-03-30 |
| Q3 | Linearity — constraint or design? | v1 constraint — open to future branching | 2026-03-30 |
| Q4 | Default flow without Transition Rules? | Auto-advance on success; halt on failure | 2026-03-30 |
| Q5 | Task Card schema? | `title` + `description` (markdown). Free-form in v1. | 2026-03-30 |
| Q6 | Transition Rule conditions? | Outcome-only in v1 (success/failure/cancelled/any). Agentic + hook conditions documented as v2 direction. | 2026-03-30 |
| Q7 | Auth standard? | Bearer token required; token issuance is server's business. Flagged as WIP in spec. | 2026-03-30 |
| Q8 | Polling latency? | Short-poll as required baseline; SSE as optional extension | 2026-03-30 |
| Q9 | Completion signal? | Process exit code + defined outcome schema. Operator elicitation method is implementation detail. | 2026-03-30 |
| Q10 | Label matching semantics? | Client's labels must be a superset of Station's required labels | 2026-03-30 |
| Q11 | Multiple Agent Definitions per Station? | No — one Agent Definition per Station in v1 | 2026-03-30 |
| Q12 | Station output passing? | Assembly Line Repository only | 2026-03-30 |

---

## Decisions Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-03-30 | Steps inside an Assembly Line → **Stations** | Fits factory metaphor; already used in vibecheck |
| 2026-03-30 | Flow control rules → **Transition Rules** | Clean, precise; maps to manufacturing concept |
| 2026-03-30 | Generic client role → **Client** (alias: Runner) | Distinguishes client (infra) from agent (AI); 1:N possible |
| 2026-03-30 | Folder location → `assembly-line/` at repo root | Mirrors tools-registry/ precedent |
| 2026-03-30 | Wire protocol → HTTP short-poll + Bearer token | Confirmed from pks-cli C# source |
| 2026-03-30 | Current scope → Linear assembly lines only | Will revisit branching in a future version |
| 2026-03-30 | Protocol name → **Assembly Line Protocol (ALP)** | Parallels MCP naming; clean acronym |
| 2026-03-30 | Stage Repository → **Assembly Line Repository** | Renamed to reflect that it belongs to the Assembly Line, not a generic staging area |
| 2026-03-30 | Transition conditions v1 → outcome-only | Matches current implementation; agentic conditions reserved for v2 |
| 2026-03-30 | Agentic conditions concept → document as v2 direction | Two types: agentic (LLM evaluator) + hook (script). Blocked task returns to current station with feedback. |
