# Agent Governance Conformance Spec

**v1 · Trust Plane–mapped**

**Scope:** Any system that runs agents on behalf of a human — internal tooling,
customer-facing products, or agents leased to a third party.
**Framework:** The Trust Plane (Trevor Bowman) — aligned to EU AI Act /
ISO 42001 / NIST AI RMF.
**Reference implementations cited:** `multica-ai/multica` (agent platform;
source of most Layer 1 mechanisms below) and `kortix-ai/suna` (§1.7,
capability-split propose/commit). Both public.

> Trust Plane's three objectives: **Govern the Runtime · Observe & Evaluate · Prove & Assure.**
> Its axiom: *"Not just protecting agents. Proving they act correctly."*

This spec turns those objectives into implementable requirements. Where a
production system has already solved a control, the concrete mechanism is cited
rather than re-derived. Where none has — the Trust Ledger — the spec says so
plainly instead of pretending.

Requirement keywords: **MUST** = conformance gate. **SHOULD** = adopt unless
there is a stated reason. **LATER** = deferred, tracked, not blocking v1.

Each requirement maps to a control in `01-assessment.md`.

---

## 0. Master mapping

| Trust Plane layer | Required controls | Mechanism | Typical status |
|---|---|---|---|
| **L1 Runtime Governance** | Human approval, agent identity, policy enforcement, tool/MCP authz, delegation controls, cost attribution & quotas | `permission_mode` deny-by-default · invocation allow-list · toolkit allowlist · `originator_user_id` chain · `accountable_user_id` · `attribution_fail_closed` | **Fully specifiable — primary v1 target** |
| **L2 Observability & Evaluation** | Traces, metrics, monitoring, telemetry, red teaming, evals, risk scoring | Unified `Message` stream · usage rollups · activity log · watchdogs · `failure_reason` classifier | **Partial industry-wide** — telemetry solved, evals/red-teaming/risk-scoring are open gaps |
| **L3 Evidence** | Prompt lineage, tool-call records, human approvals, artifacts, outcome validation, signed receipts, provenance graph | `originator_source` + evidence + lineage · source-task chain · `acceptance_criteria` · `result JSONB` · `parent_task_id` | **Partial** — provenance solved, **signed receipts absent** |
| **L4 Execution Runtime** | Agents, skills, models, memory, MCP, tools, workflows — isolation, least privilege, containment | Runtime + `Backend` adapters · skill entities · runtime profile · MCP overlay · `max_concurrent_tasks` · watchdog triad · coalescing index | **Specified below** |
| **Trust Ledger** | Immutable records, signed events, attestations, audit packages | Append-only-by-convention logs + DB CHECK invariants | **GAP — not immutable, not signed.** See §6 |

---

## 1. Layer 1 — Runtime Governance

*"Define who can do what, under which conditions, and at what cost."*

### 1.1 Agent identity & invocation authorization → **A1, A2, A3, A4**

- **MUST: deny-by-default.** Every agent carries `permission_mode ∈ {private, public_to}`,
  defaulting to `private` (owner-only invocation).
- **MUST: admin does not bypass.** A workspace or org admin **MUST NOT** inherit
  the right to invoke a private agent. *Rationale — this is a real
  privilege-escalation hole found and closed in production: an admin could
  invoke a user's private agent and, through that agent's connected OAuth apps,
  read their mailbox.* **An agent holding delegated credentials is a
  credential; treat invocation as credential use.**
- **MUST: explicit allow-list for sharing.** `public_to` resolves against
  stacking, OR-matched targets (`workspace` | `member` | `team`). No implicit
  grants.
- **SHOULD: authorization reads one source.** Legacy or derived fields (e.g. a
  `visibility` flag) may exist for display but **MUST NOT** be an authorization
  source. One gate function (`canInvokeAgent`), one input.

### 1.2 Delegation controls (agent-to-agent) → **B1, B2**

- **MUST: authorization propagates, it does not regenerate.** When agent A
  spawns work for agent B, the originating human's authority propagates down the
  chain. A fan-out **MUST NOT** acquire authority its originator lacked.
- **MUST: "no human" is representable.** `originator_user_id` is legitimately
  NULL for autopilot and system runs. NULL means *"this run carries no human
  permission"* — it is **not** a bug to be backfilled.

### 1.3 The authz/accountability split (load-bearing) → **B3, B4, B5**

Two columns, never one. Collapsing them is a latent security hazard.

| | `originator_user_id` | `accountable_user_id` |
|---|---|---|
| Question | *Whose permission is this run acting under?* | *Who do we bill / blame / show?* |
| Kind | **Authorization input** | **Audit / cost output** |
| Nullable | Yes — NULL = no human lent authority | Degrades: precise human → rule_owner → owner_fallback |
| Rule | **MUST stay unforgeable** | **MUST NOT ever grant permission** |

- **MUST:** `originator_user_id` is never defaulted, inferred, or "helpfully"
  filled. *Rationale: an owner-can-invoke-own-agent branch means a forged owner
  turns every unattributed run into a fully-privileged one — a fail-open
  escalation.*
- **MUST: enforce the one-way invariant in the database**, not only in app code:

  ```sql
  -- originator IS NOT NULL  ⟹  accountable = originator
  CHECK (originator_user_id IS NULL OR accountable_user_id = originator_user_id)
  ```

  *Rationale: in the reference implementation this constraint was added only
  after an unrelated comment-coalescing feature silently routed around the
  app-layer chokepoint and mis-attributed audited runs.* **The guardrail that
  matters is the one a future feature cannot route around.** Add `NOT VALID` for
  a metadata-only change on hot tables; it still enforces on every new write.

### 1.4 Human approval & oversight → **C3, C4**

- **MUST: fail-closed is available.** A per-workspace
  `attribution_fail_closed BOOLEAN DEFAULT FALSE`. When TRUE, a run that
  resolves to **no precise accountable human is refused at enqueue** rather than
  degrading to owner_fallback. *Better to block an unattributable run than run
  it anonymously.*
- **MUST for any third-party-facing deployment:** `attribution_fail_closed = TRUE`.
  Any agent product leased to or operated for a client runs fail-closed.
  Non-negotiable — this is the difference between a demo and something you can
  put your name on.
- **SHOULD:** irreversible or outward-facing actions (send, publish, pay,
  delete) require explicit human approval recorded as evidence (see §3),
  regardless of invocation rights.

### 1.5 Tool / MCP authorization → **C1, C2**

- **MUST: per-agent tool allow-list.** An agent mounts only an explicit set of
  toolkits and MCP servers, intersected against the owner's active connections.
- **MUST: sharing an agent shares its tools — say so.** When a `public_to` agent
  runs under the *owner's* connections, anyone permitted to invoke it can act
  through the owner's accounts. This **MUST** be surfaced at share time, not
  buried.

### 1.6 Cost attribution & quotas → **D1, D2, D4**

- **MUST: every run is attributable to an agent, a project, and a model** for
  cost purposes (see §2.2 for the telemetry shape).
- **MUST: fan-out is structurally capped, not just policy-capped.** At minimum a
  partial UNIQUE index guaranteeing at most one in-flight task per work item:

  ```sql
  CREATE UNIQUE INDEX one_pending_task_per_issue
    ON agent_task_queue (issue_id) WHERE status IN ('queued','dispatched');
  ```

  *Rationale: the publicly reported industry blow-ups — $8K to $47K — were
  fan-out, not single-run cost. A DB constraint refuses the storm with zero app
  logic and cannot be bypassed by a future feature.*
- **MUST:** operational guardrails stand — subagent parallelism capped (≤ 3 is a
  reasonable default), no nested subagent chains, no unattended overnight runs,
  turn caps on anything long-running.

### 1.7 Propose/commit capability split → **C5**

- **SHOULD: split "propose" and "commit-to-shared-state" into distinct
  capabilities**, not one write permission. The git-native pattern from
  `kortix-ai/suna`: a session's scoped token can carry `project.cr.open` without
  `project.cr.merge` — the agent can push work and open a change request but is
  structurally unable to land it on the default branch. **The boundary is
  enforced by the token, not by an instruction the agent could ignore.**
- *Rationale:* §1.4's "human approval" is typically enforced by convention —
  policy docs, prose, a request in chat. That holds for a trusted
  single-operator agent and is not real enforcement for a client-facing or
  less-trusted one. A capability split makes the boundary structural instead of
  aspirational.
- **Partially closes the §6 gap #3 (Outcome → Approval).** If the commit action
  is built this way, it doubles as the approval record for free: who committed,
  when, against what artifact. Adopt when an agent is given direct write access
  to a shared branch or target.

---

## 2. Layer 2 — Observability & Evaluation

*"Observe everything that matters and evaluate quality and risk."*

### 2.1 Traces → **E1**

- **MUST: one normalized event vocabulary across every runtime.** A unified
  `Message` type is the model: `text | thinking | tool-use | tool-result |
  status | error | log`, with tool name, call id, input, and output captured per
  call. Heterogeneous CLIs normalize into it at the adapter boundary.
- **SHOULD:** tool-call records are the trace *and* the L3 evidence — capture
  once, use twice.

### 2.2 Metrics & telemetry (cost) → **D2, D3**

- **MUST: token usage recorded per `(task, provider, model)`**, with
  `input / output / cache_read / cache_write` tracked **separately**. *Cache
  tokens dominate real cost on frontier models; a single "tokens" number hides
  the thing you actually need.*
- **SHOULD: pre-grouped rollups, UTC-bucketed hourly**, keyed by
  `(bucket_hour, workspace, runtime, agent, project, provider, model)`. Hourly
  UTC is timezone-neutral — any viewer timezone applies at query time without
  re-materializing. Every cost view becomes a pre-grouped read, not a raw scan.
- **This upgrades you from a blunt turn cap to answering "which agent or project
  burned the budget."**

### 2.3 Monitoring & liveness → **E2, E3**

- **MUST: watchdog-based liveness, not a single blunt timeout.** A three-timer
  model:
  - `IdleWatchdogTimeout` — no-message watchdog
  - `HandshakeTimeout` — bounds startup RPCs
  - `SemanticInactivityTimeout` — semantic stall detection

  A zero per-run wall-clock timeout means *"no deadline; rely on the inactivity
  watchdog"* — so a long-but-actively-progressing run is never killed merely for
  running long, while a hung one dies fast. **This is strictly better than a turn
  cap for runaway protection** — it distinguishes *stuck* from *busy*.
- **SHOULD: classify failures, don't just record them.**
  `failure_reason ∈ {agent_error, timeout, runtime_offline, runtime_recovery, manual}`,
  and the retry path **reads** it: a `runtime_offline` orphan is worth
  re-running; an `agent_error` usually is not. Retry policy is driven by
  classification, never blind.

### 2.4 Evaluation, red teaming, risk scoring — **GAP**

- **LATER:** No production agent platform surveyed offers an eval harness,
  red-teaming, or risk scoring as a built-in. Pre-launch surface sweeps are the
  common substitute and are not continuous evaluation. Until closed, do not
  claim L2 conformance — claim *telemetry* conformance.

---

## 3. Layer 3 — Evidence → **F1–F5**

*"Build tamper-proof evidence for every decision, action, and outcome."*

- **MUST: provenance is stamped, not reconstructed.** Every run records
  `originator_source` (how the run was triggered), its evidence, and its lineage
  at enqueue time. Reconstructing provenance after the fact is not evidence.
- **MUST: retry lineage is explicit.** `parent_task_id` back-pointer plus
  `attempt` / `max_attempts`, so a re-run is provably a re-run rather than a new,
  unexplained action.
- **MUST: outcome validation is machine-checkable.** Work items carry
  `acceptance_criteria` as **structured JSONB, not prose.** *An agent cannot
  self-verify against a paragraph; it can against a checklist.* This is
  simultaneously the L3 outcome-validation control and the thing that makes
  agent work verifiable at all. Pair with `context_refs` (what the agent should
  read first).
- **MUST: artifacts and outputs are retained** on the task record
  (`result JSONB`, `error TEXT`), distinct from the human-facing status.
- **SHOULD: the provenance graph is queryable** — the
  human→agent→task→comment→spawned-task chain.
- **Signed receipts — GAP.** See §6.

### 3.1 Lineage chain conformance

Trust Plane's chain:
**Human → Agent → Skill → Prompt → Policy → Model → Tool → Artifact → Outcome → Approval.**
*"Every step creates evidence. Every edge creates traceability."*

Typical coverage for a well-built platform:

| Edge | Covered by | Status |
|---|---|---|
| Human → Agent | `originator_user_id` + `accountable_user_id` | Achievable |
| Agent → Skill | `agent_skill` (+ `enabled`) | Achievable |
| Skill → Prompt | Skill `content` + injected instructions | Achievable |
| Prompt → Policy | `canInvokeAgent` gate + `permission_mode` | Achievable |
| Policy → Model | `task_usage.model`, runtime profile, thinking level | Achievable |
| Model → Tool | `tool-use` / `tool-result` messages + toolkit allowlist | Achievable |
| Tool → Artifact | `result JSONB`, attachments | Achievable |
| Artifact → Outcome | `acceptance_criteria` validation | **Partial** — criteria exist; automated validation is the gap |
| Outcome → Approval | — | **GAP** — no approval record type in common use |

---

## 4. Layer 4 — Execution Runtime

*"Run agents and skills securely with isolation, least privilege, and containment."*

- **MUST: least privilege at the runtime boundary** — an agent mounts only its
  allow-listed tools (§1.5) and only the skills bound to it, gated by an
  `enabled` flag.
- **MUST: containment.** `max_concurrent_tasks` per agent, the coalescing index
  (§1.6), and the watchdogs (§2.3) are the containment triad. All three, not
  one.
- **SHOULD: one `Backend` interface, many adapters.**
  `Execute(ctx, prompt, opts) → Session{Messages, Result}`. Adapters normalize
  each CLI into the unified event vocabulary. Capability differences degrade via
  **silent fall-through** (an adapter lacking a thinking-level control ignores it
  rather than erroring) so new runtimes land incrementally without breaking
  existing ones.
- **SHOULD: the daemon pulls, the server does not push.** Work is claimed with
  `FOR UPDATE SKIP LOCKED` plus a claim token. An offline runtime simply doesn't
  claim; work waits. No broker, no always-on assumption, no double-dispatch.
- **SHOULD: session continuity is honest.** Distinguish *"intended to resume"*
  from *"here's the session id"*; when a resume is rejected and a retry lands on
  a fresh thread, **surface a continuity notice rather than silently starting
  over.** Silent amnesia is an evidence failure, not just a UX wart.
- **SHOULD: skills are first-class, portable entities** —
  `skill(name, description, content, config)` + `skill_file(path, content)` +
  an agent↔skill many-to-many, parsing standard `SKILL.md` YAML frontmatter.

---

## 5. Deployment posture by exposure

Conformance requirements scale with who the agent acts for. Find your row.

| Exposure | Fail-closed | Required floor | Notes |
|---|---|---|---|
| **Single-operator, internal** (agent acts only for its owner) | N/A | §1.1, §1.5, §2.3 | The priority is credential scope. If the agent holds mail/calendar/drive/chat connections, owner-only invocation and per-agent toolkit allow-lists are the whole game. **The moment it becomes multi-user or exposed via a bot interface, all of L1 becomes MUST.** |
| **Multi-user, internal** (colleagues can invoke) | SHOULD | Full L1 | §1.3's authz/accountability split becomes load-bearing the day a second human can trigger a run. |
| **Customer-facing product** | SHOULD → MUST | Full L1 + L3 | L1 and L3 are the differentiators for anything sold as governed. Governance *is* the product. |
| **Leased / operated for a client** | **MUST = TRUE** | Full L1 + L3 | Every action traces to a named accountable human. This is the assurance story you sell, and the one an enterprise buyer's security review will test. |
| **Regulated or audited** | **MUST = TRUE** | Full L1 + L3 + Trust Ledger (§6) | Without tamper-evidence you cannot make an evidentiary claim. See the conformance honesty rule. |

**Recommended port order** for an existing system moving up a row:
`acceptance_criteria` / `context_refs` (§3) → dual state machine →
`SKIP LOCKED` + coalescing index (§1.6) → authz/accountability split (§1.3).

---

## 6. Gaps — stated, not papered over

These are open across the industry, not defects in one implementation. Naming
them is part of conformance.

1. **Trust Ledger (immutability + signing).** Activity logs are typically
   append-only *by convention*, not enforcement: no hash chain, no signed
   events, no attestations, no exportable audit package. **Anything claiming EU
   AI Act / ISO 42001 evidentiary weight needs this.** The honest claim without
   it: "attributable and traceable," **not** "tamper-proof."
2. **Evaluation / red teaming / risk scoring (L2).** No harness in common use. A
   pre-launch surface sweep is not continuous agent evaluation.
3. **Outcome → Approval edge (L3).** No approval record type. Human approvals
   happen in chat and leave no structured evidence. §1.7 sketches a mechanism
   (commit-capability held only by the human) that closes this as a byproduct.
4. **Automated outcome validation.** `acceptance_criteria` can be structured, but
   nothing yet *checks* the agent's output against it automatically.

**Conformance honesty rule:** claim only what is built. A realistic strong
posture today is **L1 enforced, L4 strong, L2 telemetry-only, L3 partial, Trust
Ledger absent.** Do not let a sales deck say otherwise. The buyer's security
reviewer will find the gap, and finding it stated is worth more than finding it
hidden.

---

## 7. Conformance checklist

A system claiming conformance with this spec satisfies every **MUST**:

- [ ] Agents default to `private` (owner-only invocation)
- [ ] Admin/org role does **not** bypass private invocation
- [ ] Sharing is an explicit, stacking allow-list; authorization reads exactly one source
- [ ] Delegated authority propagates and never escalates; "no human" (NULL) is representable
- [ ] `originator_user_id` (authz) and `accountable_user_id` (audit) are separate columns
- [ ] `originator_user_id` is never defaulted or inferred
- [ ] The one-way invariant is enforced by a **database CHECK**, not app code alone
- [ ] `attribution_fail_closed` exists; **TRUE for any third-party-facing deployment**
- [ ] Per-agent tool/MCP allow-list; shared-agent credential exposure disclosed at share time
- [ ] Every run attributable to agent + project + model
- [ ] Fan-out capped by a DB constraint (partial UNIQUE index), not policy alone
- [ ] Subagent parallelism capped; no nested chains; no unattended overnight runs
- [ ] One normalized event vocabulary across runtimes
- [ ] Token usage per `(task, provider, model)` with cache tokens separate
- [ ] Watchdog-based liveness (idle / handshake / semantic), not a single blunt timeout
- [ ] Failure reasons classified; retry policy reads the classifier
- [ ] Provenance stamped at enqueue; retry lineage explicit
- [ ] `acceptance_criteria` structured (JSONB), not prose
- [ ] Artifacts/outputs retained separately from human-facing status
- [ ] Gaps in §6 disclosed rather than implied as solved
