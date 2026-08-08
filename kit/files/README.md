# Agent Governance Starter Kit

**For engineering leaders running AI agents in production.**

You have agents doing real work — reading mail, writing to databases, spawning
other agents, spending money. Somewhere between "it works on my machine" and
"a customer's security reviewer is asking questions," there is a gap. This kit
tells you exactly how wide yours is.

---

## What's inside

| File | What it does | Time |
|---|---|---|
| **`01-assessment.md`** | 22-control scored self-assessment. Produces a band (1–5) and a prioritised fix list. Six controls are hard gates — fail one and your band is capped regardless of total. | ~45 min |
| **`02-conformance-spec.md`** | The implementable requirement set behind every control. MUST/SHOULD/LATER, with SQL, schema shapes, and the rationale for each. Every requirement maps back to an assessment control. | Reference |

Run the assessment first. Read the spec for the controls you failed.

---

## Why this is not another checklist

Three things make this different from a generic AI-governance PDF:

**1. It is mapped to a real framework, not invented.**
Every control traces to the Trust Plane (Trevor Bowman) — the layered model
aligned to EU AI Act, ISO 42001, and NIST AI RMF. You are not scoring against
one consultant's opinion.

**2. The mechanisms are extracted from production systems, not theory.**
The database constraints, the watchdog triad, the authz/accountability split —
these come from open-source agent platforms that hit the bugs first. Where a
control exists because a real system shipped a privilege-escalation hole and
closed it, the rationale says so.

**3. It refuses to average away the controls that actually breach.**
Most maturity models let a high score in observability paper over a fail-open
authorization path. This one doesn't. Six controls are gates. A fail-open
escalation is not offset by good telemetry.

---

## The uncomfortable finding

Most teams running production agents score **Band 2 — Documented**, and most
believe they are Band 3 or 4.

The gap is almost always the same: controls that live in prose — a policy doc,
a README, an instruction in the system prompt — rather than in constraints and
gates. A model can ignore an instruction. A future feature can route around an
application-layer check without ever knowing it existed.

It cannot route around a `CHECK` constraint.

If your assessment lands in Band 1 or 2 and you have client-facing or regulated
exposure, you do not have a documentation problem.

---

## What this kit does not do

Stated plainly, because the whole kit runs on a conformance honesty rule:

- It does not implement anything. It tells you what to build and why.
- It does not cover model safety, alignment, or content filtering. This is
  runtime governance — authorization, cost, evidence.
- It does not close the Trust Ledger gap (§6). Nothing on the market currently
  does. If you need signed, tamper-evident attestations for a regulator, the
  spec tells you that honestly rather than selling you a checkbox.

---

## After the assessment

The fix list at the end of `01-assessment.md` is ordered by blast radius, not by
score. Work it in that order and the highest-risk paths close first.

If you would rather not run it yourself — or you scored Band 1–2 and need the
remediation done against a live system — that is what the **AI Governance
Audit** is for: three weeks, discovery → Trust Plane mapping → gap report →
remediation roadmap, run against your actual codebase rather than a
self-report.

Details: **offstageai.com**

---

*Offstage Consultants · Agent Governance Starter Kit v1*
