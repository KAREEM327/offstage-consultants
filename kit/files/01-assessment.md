# Agent Governance Self-Assessment

**Scored diagnostic · 22 controls · ~45 minutes**

You are running agents that act on behalf of humans. This tells you where that
is safe, where it is not, and what to fix first.

Score honestly. A generous score produces a comfortable report and an
uncontrolled system. The point of the exercise is the failures.

---

## How to score

Each control gets one of three values:

| Score | Meaning |
|---|---|
| **2** | **Enforced.** A mechanism refuses the bad case. Code, a constraint, a config gate. |
| **1** | **Documented.** The rule exists in prose — a policy, a README, an instruction to the agent. Nothing stops a violation. |
| **0** | **Absent.** Not addressed. |

**The 1 is the score most systems land on and the one most teams overstate as a 2.**
Ask: *if a new feature shipped next week that violated this, what would stop it?*
If the answer is "someone would notice in review," that is a 1.

Six controls are marked **[GATE]**. They are fail-open escalation paths — a
violation grants privilege rather than merely failing to record it. Any GATE
scored below 2 caps your band regardless of total score. This is deliberate.
Averages hide the controls that actually breach.

---

## Section A — Invocation authorization
*Who is allowed to make this agent run?*

**A1 [GATE] — Deny-by-default invocation.**
Every agent has an explicit permission mode defaulting to owner-only. A new
agent is private until someone deliberately shares it.
`[ ] 0  [ ] 1  [ ] 2`

**A2 [GATE] — Admin does not bypass.**
A workspace, org, or platform admin cannot invoke a private agent by virtue of
their role.
*Why this is a gate: an agent holding delegated OAuth credentials is a
credential. If an admin can invoke your private agent, the admin can read your
mailbox through it. Invocation is credential use.*
`[ ] 0  [ ] 1  [ ] 2`

**A3 — Sharing is an explicit allow-list.**
Sharing resolves against enumerated targets (user, team, workspace). No
implicit grants, no "everyone in the org by default."
`[ ] 0  [ ] 1  [ ] 2`

**A4 — Authorization reads exactly one source.**
One gate function, one input field. Display flags and legacy fields exist for
UI but are never consulted for an authorization decision.
`[ ] 0  [ ] 1  [ ] 2`

---

## Section B — Delegation and accountability
*When agent A spawns agent B, whose authority is B running under?*

**B1 [GATE] — Authority propagates, never regenerates.**
When an agent spawns downstream work, the originating human's authority is
carried down the chain. A fan-out cannot acquire permission its originator
lacked.
`[ ] 0  [ ] 1  [ ] 2`

**B2 — "No human" is representable.**
Autopilot, cron, and system-triggered runs can legitimately record *no*
originating human. That state means "this run carries no human permission" and
is not treated as missing data to be backfilled.
`[ ] 0  [ ] 1  [ ] 2`

**B3 [GATE] — Authorization and accountability are separate fields.**
Two distinct records: *whose permission is this acting under* (authorization
input) and *who do we bill and name* (audit output). They are not one column.
*Why this is a gate: collapse them and the billing field becomes an
authorization source. A run attributed to the owner for cost purposes becomes a
run running with owner privileges.*
`[ ] 0  [ ] 1  [ ] 2`

**B4 [GATE] — The authorization field is never defaulted or inferred.**
No code path fills in the originating human "helpfully." No owner-can-invoke-
own-agent shortcut that a forged or absent value can ride.
*Why this is a gate: this is the specific mechanism that turns every
unattributed run into a fully-privileged one.*
`[ ] 0  [ ] 1  [ ] 2`

**B5 — The one-way invariant is enforced at the storage layer.**
The rule "if there is an originating human, the accountable party is that same
human" is a database constraint, not application code.

```sql
CHECK (originator_user_id IS NULL OR accountable_user_id = originator_user_id)
```

*The guardrail that matters is the one a future feature cannot route around.
Application-layer chokepoints get bypassed by unrelated features that had no
idea the chokepoint existed.*
`[ ] 0  [ ] 1  [ ] 2`

---

## Section C — Tool access and human oversight
*What can the agent reach, and what requires a human first?*

**C1 — Per-agent tool allow-list.**
An agent mounts an explicit set of tools and MCP servers, intersected against
what the owner has actually connected. Not "all available tools."
`[ ] 0  [ ] 1  [ ] 2`

**C2 — Shared-agent credential exposure is disclosed at share time.**
When a shared agent runs under the owner's connections, anyone who can invoke
it can act through the owner's accounts. This is surfaced in the sharing flow,
not buried in documentation.
`[ ] 0  [ ] 1  [ ] 2`

**C3 — Irreversible actions require recorded human approval.**
Send, publish, pay, delete, deploy. Approval is captured as a record, not as a
message in a chat log.
`[ ] 0  [ ] 1  [ ] 2`

**C4 — Fail-closed mode exists.**
A setting under which a run that cannot resolve to a named accountable human is
**refused at enqueue** rather than degrading to a fallback owner.
*Required TRUE for any deployment where the agent acts for someone other than
its operator.*
`[ ] 0  [ ] 1  [ ] 2`

**C5 — Propose and commit are separate capabilities.**
The agent can produce work and open it for review, but is structurally unable
to land it on shared state. The boundary is held by the token or permission,
not by an instruction the agent could ignore.
`[ ] 0  [ ] 1  [ ] 2`

---

## Section D — Cost containment
*What stops a runaway?*

**D1 [GATE] — Fan-out is capped by a constraint, not a policy.**
At minimum, a uniqueness guarantee that at most one task is in flight per work
item.

```sql
CREATE UNIQUE INDEX one_pending_task_per_issue
  ON agent_task_queue (issue_id) WHERE status IN ('queued','dispatched');
```

*Why this is a gate: the publicly reported agent cost blow-ups — $8K to $47K —
were fan-out, not single-run cost. A policy in a prompt does not refuse a
storm. A database constraint does, with zero application logic, and cannot be
bypassed by a future feature.*
`[ ] 0  [ ] 1  [ ] 2`

**D2 — Every run is attributable to an agent, a project, and a model.**
`[ ] 0  [ ] 1  [ ] 2`

**D3 — Token usage is recorded with cache tokens tracked separately.**
Input, output, cache-read, and cache-write are four numbers, not one.
*Cache tokens dominate real cost on frontier models. A single "tokens" figure
hides the line item you need.*
`[ ] 0  [ ] 1  [ ] 2`

**D4 — Concurrency and nesting are bounded.**
A cap on simultaneous subagents, no nested spawn chains, no unattended
long-running jobs without a turn cap and a human check-in.
`[ ] 0  [ ] 1  [ ] 2`

---

## Section E — Observability
*Can you see what happened?*

**E1 — One normalized event vocabulary across every runtime.**
Text, reasoning, tool-use, tool-result, status, error, log — captured
identically whether the run came from one CLI or another. Heterogeneous
runtimes normalize at the adapter boundary.
`[ ] 0  [ ] 1  [ ] 2`

**E2 — Liveness uses watchdogs, not a single wall-clock timeout.**
Separate timers for idle, startup handshake, and semantic stall. A long
run that is actively progressing is not killed for running long; a hung one
dies fast.
*A blunt timeout cannot distinguish stuck from busy. Watchdogs can.*
`[ ] 0  [ ] 1  [ ] 2`

**E3 — Failures are classified, and retry policy reads the classification.**
An infrastructure-offline orphan is worth re-running. An agent error usually is
not. Retry is never blind.
`[ ] 0  [ ] 1  [ ] 2`

---

## Section F — Evidence
*Can you prove what happened?*

**F1 — Provenance is stamped at enqueue, not reconstructed.**
How the run was triggered, under whose authority, and against what context is
recorded when the run is created. *Provenance reassembled after the fact from
logs is not evidence.*
`[ ] 0  [ ] 1  [ ] 2`

**F2 — Retry lineage is explicit.**
A re-run points at its parent and carries an attempt count, so it is provably a
re-run rather than a new unexplained action.
`[ ] 0  [ ] 1  [ ] 2`

**F3 — Acceptance criteria are structured, not prose.**
Work items carry machine-checkable criteria. *An agent cannot self-verify
against a paragraph. It can against a checklist.*
`[ ] 0  [ ] 1  [ ] 2`

**F4 — Artifacts and raw outputs are retained separately from status.**
The human-facing "done" is not the only surviving record of what was produced.
`[ ] 0  [ ] 1  [ ] 2`

**F5 — Records are tamper-evident.**
Hash chaining, signing, or an equivalent that makes after-the-fact editing
detectable. Append-only *by convention* scores 1, not 2.
`[ ] 0  [ ] 1  [ ] 2`

---

## Scoring

**Total possible: 44** (22 controls × 2)

Add your scores. Then apply the gates.

### Gate check

The six GATE controls: **A1, A2, B1, B3, B4, D1**

| Lowest GATE score | Your band is capped at |
|---|---|
| Any GATE at **0** | **Band 1 — Uncontrolled** (regardless of total) |
| Any GATE at **1** | **Band 2 — Documented** (regardless of total) |
| All GATEs at **2** | Uncapped — use your total |

### Bands

| Total | Band | What it means |
|---|---|---|
| 0–14 | **1 — Uncontrolled** | Agents act with effectively unbounded authority. You cannot answer "who authorized this" or "what stopped it." Do not deploy on behalf of a third party. |
| 15–26 | **2 — Documented** | The rules exist and the team mostly follows them. Nothing enforces them. This is the band where a single unrelated feature silently routes around your controls. |
| 27–36 | **3 — Enforced** | Core authorization and cost paths are mechanically enforced. Evidence is partial. Defensible internally; not yet defensible to an auditor. |
| 37–42 | **4 — Attributable** | Every action traces to a named accountable human through enforced mechanisms. Honest claim: *attributable and traceable.* |
| 43–44 | **5 — Assured** | Attributable plus tamper-evident. The only band from which evidentiary claims to a regulator or enterprise buyer survive scrutiny. |

---

## The honesty rule

Claim only the band you scored, and name your gaps rather than implying they
are solved.

Most teams running production agents in 2026 land in **Band 2**, and most
believe they are in Band 3 or 4. The difference is almost always the same
thing: controls that live in prose and instructions rather than in constraints
and gates. A model can ignore an instruction. It cannot ignore a `CHECK`.

If your assessment puts you in Band 1 or 2 and you have client-facing or
regulated exposure, the gap is not a documentation problem.

---

## What to fix first

Work in this order. It is deliberately not "lowest score first" — it is
blast-radius first.

1. **Any GATE below 2.** These are privilege-escalation paths. Nothing else
   matters until they are closed.
2. **D1 (fan-out constraint).** Cheapest control-to-risk ratio on the list. One
   index. Prevents the failure mode that has produced the largest publicly
   reported losses.
3. **B5 (storage-layer invariant).** Converts your most important rule from
   convention into enforcement. Add it `NOT VALID` on hot tables — it is a
   metadata-only change and still enforces every new write.
4. **C4 (fail-closed).** Required before any third-party-facing deployment.
5. **F1–F3 (evidence).** These determine whether you can sell to an enterprise
   buyer or survive an audit. Structured acceptance criteria (F3) also
   unlock automated outcome validation later.
6. **Everything else**, by score.

---

*Continue to `02-conformance-spec.md` for the implementable requirement set
behind each control.*
