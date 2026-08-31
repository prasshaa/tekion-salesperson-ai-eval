# PRD — Confidence-Routed Evaluation Layer for Salesperson AI

**Status:** Concept / case study
**Author:** Prathyusha Vedula
**Product area:** Tekion CRM AI
**Date:** August 2026

---

## 1. Background — What Salesperson AI Is Today

Salesperson AI is the customer-facing agent inside Tekion's **CRM AI** suite (nav path: `tekion.com/crm-ai`), built directly into Automotive Retail Cloud (ARC) — Tekion's DMS/CRM/service/accounting platform. It is not a bolt-on chatbot; it reads and writes to the same live inventory, deal, and customer-history data every other ARC workflow uses.

**What it does today, per Tekion's own product page:**
- Responds to every inbound lead inquiry (text or email) within seconds, 24/7 — including nights, weekends, holidays.
- Answers real buyer questions on pricing, availability, and vehicle features using **live inventory and deal context**, not templated replies.
- Runs a **systematic 30-day automated follow-up sequence** across active leads, unsold visits, missed appointments, and post-sale nurture — same cadence for every lead, not dependent on which rep it was assigned to.
- Books appointments directly into the CRM calendar.
- Performs **intelligent handoff**: when a conversation needs a human, it generates a full summary and next-action recommendation so the rep "inherits context, not a cold thread."
- Is **dealer-configurable** — FAQs, operating hours, tone are all set per dealership, and any lead can be disengaged from AI with one click.
- Logs every AI interaction in real time inside Lead Details, the Communication Hub, and the Chat Modal — full visibility, not a black box.

**This makes Salesperson AI meaningfully more autonomous than it might sound.** It isn't drafting a reply for a rep to approve — it is independently composing and sending messages to real customers, and independently deciding when a 30-day follow-up sequence continues, pauses, or escalates. The rep is in the loop for *oversight and override*, not for *approval of each message*. That's the right design for the product's core value prop (speed beats every competing dealer's slower human response), but it's also exactly the design that makes evaluation and confidence-routing a first-order product concern rather than a nice-to-have.

## 2. Problem Statement

Two things are true at once:
1. Response speed is explicitly the product's differentiator — Tekion's own marketing frames "the customer who buys from the first dealer to respond" as the prize, and every added second of human review erodes that.
2. The agent sends unreviewed messages that can contain **specific, falsifiable claims** — price, availability, financing eligibility — to real customers, with real legal/compliance/brand exposure if it gets one wrong (an unauthorized approval guarantee, a stale price, a mis-scoped promise).

Today's public materials show a **static configuration layer** (FAQs, tone, hours, disengage-toggle) and a **downstream audit trail** (everything is logged). What isn't visible is a **pre-send quality gate** — some way of knowing, per message, "how confident should we be in this response, and does it clear the bar to go out with zero human review?"

Without that layer, the product has exactly two failure-prone defaults: send everything (fast, but the tail-risk message goes out unreviewed), or route everything through a human for review (safe, but destroys the speed advantage that is the entire point of the product). Neither is the actual answer — the answer is a **calibrated, evidence-based routing decision made per message**, which is what this PRD proposes.

## 3. Goals

- Reduce the rate of high-risk agent outputs (unauthorized promises, unverifiable claims, missed escalations) reaching a customer with zero human visibility, **without** materially increasing average response latency for the large majority of routine, low-risk messages.
- Give Tekion (internally) and dealers (in-product) a **quantified, auditable quality signal** per message and in aggregate — not just "the AI sent something," but "the AI sent something we can show was evaluated and how confident the system was."
- Make the routing decision **explainable and tunable per dealer**, consistent with the product's existing "dealer configurability" philosophy (FAQs/tone/hours are already dealer-set; risk tolerance should be too).

## 4. Non-Goals

- This is not a proposal to change Salesperson AI's underlying response-generation model or prompt strategy — it's a layer that sits *after* generation and *before* send.
- Not proposing to make every message human-reviewed — that would defeat the product's core value proposition and isn't what dealers are buying.
- Not attempting to eliminate hallucination/error at the generation step — that's a model/retrieval problem; this PRD treats generation as a black box and focuses on **catching and routing** what comes out of it.

## 5. Users & Stakeholders

| User | What they need from this |
|---|---|
| **Dealer sales manager** | Confidence that the AI won't damage a customer relationship or make an unauthorized commitment — and visibility when it's uncertain, so they can jump in before, not after |
| **Lead / customer** | A fast, accurate response — or a clear, fast handoff to a human when the AI genuinely doesn't know |
| **Tekion CRM AI product team** | A quantified way to say "the system is getting better" release over release, and a hard release gate that prevents shipping a regression that increases unauthorized-commitment rate |
| **Tekion Trust/Compliance** | An auditable per-message decision trail — this is also what an ISO 42001-certified AI management system is expected to demonstrate |

## 6. Proposed Solution — Overview

The first pass at this proposal put a full LLM-as-judge + self-consistency check between "draft" and "send," gated on both together. That doesn't survive contact with the product's actual constraint: judge scoring and self-consistency sampling are each a full extra model call (self-consistency needs several), typically seconds combined — and Salesperson AI's entire value proposition is a sub-second, no-human response. A gate that takes longer to run than the reply took to generate isn't something a real-time chat product can afford, and the honest version of this proposal has to design around that instead of asserting past it.

So this splits into three tiers, ordered by how cheap and reliable each one is, matched to where it actually sits in the pipeline:

**Tier 0 — Pre-generation topic routing (near-zero latency).**
Before Salesperson AI drafts anything, a lightweight classifier reads the lead's message and flags its risk category: routine (pricing, availability, scheduling) vs. high-risk (financing/approval language, legal terms, complaints, negotiation past dealer-set limits). High-risk messages skip autonomous generation entirely and go straight into the product's existing "intelligent handoff" behavior — drafted for a human, not sent directly. This is genuine, real-time prevention, and it's cheap because a topic classifier is a much smaller, faster model than a judge reasoning over full rubric + ground-truth context.

**Tier 1 — Deterministic pre-send checks (near-zero latency).**
For routine-category replies that do get autonomously generated, a small set of rule-based checks run in the send path: does the reply contain approval-guarantee language ("guaranteed," "you'll definitely be approved")? Does a quoted price or availability claim match the live inventory record (a direct lookup, not a model call)? Fast enough to sit on the critical path without meaningfully affecting response time — this catches the small number of known, specific failure patterns without needing an LLM call at all.

**Tier 2 — Async LLM-as-judge + confidence scoring (off the critical path, runs after send).**
Every autonomously-sent reply is scored by the full rubric judge and an objective confidence signal *after* it's already gone out. This tier cannot prevent a bad message from reaching a customer — it exists to catch it fast and close the loop: flag violations within minutes, pause any further autonomous follow-up in that thread pending review, log for compliance, and feed newly-discovered failure patterns back into Tier 0's classifier and Tier 1's rules over time — the same "gets smarter with every correction" pattern Accounts Payable AI already uses, applied here as a feedback loop rather than a live gate.

| Tier | Runs | Catches | Latency/cost |
|---|---|---|---|
| 0 — Topic routing | Before generation | Known high-risk categories | Near-zero — small classifier |
| 1 — Deterministic checks | In the send path | Specific known-bad patterns | Near-zero — rules/lookups, no model call |
| 2 — Judge + confidence | After send, async | Nuanced groundedness, tone, escalation misses, novel failures | Seconds, but off the critical path |

This is a narrower claim than the original version — and a more honest one: real-time **prevention** only for the cheap, known failure modes; fast **detection and remediation** for everything nuanced enough to need genuine semantic judgment, with Tier 2's findings continuously widening what Tiers 0 and 1 can catch on their own.

## 7. Requirements

### Functional
- **FR1**: A topic/risk classifier runs on every inbound lead message before generation; messages flagged high-risk (financing, legal, complaints, out-of-limit negotiation) route to human-assist drafting rather than autonomous send (Tier 0).
- **FR2**: Deterministic pattern checks (approval-guarantee language, live-inventory price/availability match) run on every autonomously-generated reply before send; any match blocks send and routes to human review (Tier 1).
- **FR3**: Every autonomously-sent reply is scored asynchronously by the full five-dimension LLM-as-judge rubric plus an objective confidence signal — this scoring does not gate the send (Tier 2).
- **FR4**: Any Tier 2 detection of a Safety-dimension violation triggers, without waiting for human pickup: an immediate alert to the assigned rep/manager, an automatic pause on further autonomous follow-up in that thread, and a logged compliance record.
- **FR5**: Tier 0's classifier and Tier 1's rule set are updated on a defined cadence using confirmed violations Tier 2 surfaces — the feedback loop is a requirement, not a side effect.
- **FR6**: Every classification, check, and async score is logged and queryable — extends the existing Communication Hub audit trail rather than replacing it.
- **FR7**: A rolling dashboard (judge/human agreement, safety violation rate, Tier 0/1 catch rate, time-to-detect, time-to-remediate) is available to the CRM AI product team and, in a lighter form, to dealers.
- **FR8**: Any change to Salesperson AI's generation prompt or underlying model, any new Tier 1 rule sourced from a Tier 2 finding, and any retrained Tier 0 classifier must pass an **offline regression gate** — evaluated against the labeled gold-set, with no regression in Safety/Escalation scores or false-positive rate — before it ships to production (see `eval-framework.md` §5.1). Nothing that changes what's live ships without first being checked against this held-out set.

### Non-functional
- **NFR1 — Latency budget (Tiers 0+1 only)**: Combined, these must add no more than ~50–150ms to the autonomous-send path — a small classifier plus rule/lookup checks, no full LLM judge call in this budget. This is the only tier with a real latency constraint.
- **NFR2 — Detection SLA (Tier 2)**: Async scoring and any resulting alert must complete within a defined window (minutes, not hours) of send — Tier 2 has no send-side latency budget, but it does have a freshness requirement, since it's the layer responsible for catching what Tiers 0/1 miss before the same error reaches another lead or an unsupervised follow-up sequence continues.
- **NFR3 — Cost budget**: Tier 2's judge + confidence pipeline is the dominant cost driver. Given Tekion's platform already runs an **LLM Gateway** for multi-provider routing and cost tracking, Tier 2 should run on a smaller/distilled model for full-traffic coverage, with a frontier-model spot-check on a 5–10% sample to catch drift.
- **NFR4 — Multi-tenant isolation**: Consistent with how Accounts Payable AI's GL-code learning is scoped per store, Tier 0/1 rule updates learned from one dealership's Tier 2 findings must not silently change another dealership's behavior without going through the same calibration discipline.
- **NFR5 — Explainability**: Every Tier 0/1/2 decision is traceable to the specific rule, pattern match, or rubric dimension that triggered it — not a single opaque score.

## 8. System Design (Architecture Summary)

```
Lead message received
        │
        ▼
┌───────────────────────────────┐
│ TIER 0 — Topic/risk classifier │   near-zero latency
│  routine   → autonomous path   │
│  high-risk → human-assist draft│
└───────────────────────────────┘
        │ (routine path)
        ▼
Salesperson AI generates draft reply
  (existing system — grounded in live inventory/deal context)
        │
        ▼
┌───────────────────────────────┐
│ TIER 1 — Deterministic checks  │   near-zero latency, no model call
│  guarantee-language pattern    │
│  price/availability vs. live   │
│  inventory lookup              │
└───────────────────────────────┘
        │
   pass │            fail → hold for human review before send
        ▼
     SEND to lead
        │
        ▼  (off critical path, async)
┌───────────────────────────────┐
│ TIER 2 — LLM-as-judge +        │   seconds, async — full traffic
│ confidence scoring             │
└───────────────────────────────┘
        │
        ▼
 violation? ──yes──▶ alert rep/manager, pause thread's autonomous
        │             follow-up, log for compliance
        no
        ▼
 feed into Tier 0/1 rule + classifier updates (periodic recalibration)
        │
        ▼
Audit log + dashboard (extends Communication Hub)
```

The key shift from an earlier version of this design: Tiers 0 and 1 are the only components with a real-time latency budget, and they're intentionally cheap and narrow — a topic classifier and a rules engine, not a semantic judge. Tier 2 does the actual nuanced evaluation, but never on the send path.

Full technical detail — the judge prompt itself, how the confidence signal is computed, why these specific methods over alternatives — is in **[`eval-framework.md`](./eval-framework.md)**, which this PRD deliberately keeps separate so the product framing here doesn't get lost in implementation detail.

## 9. Success Metrics

| Metric | What it tells us |
|---|---|
| **Safety violation rate reaching a customer** (per 1,000 autonomous sends) | The core risk metric — note this can never be driven to zero, since Tiers 0/1 are necessarily incomplete; the honest goal is minimizing it and shrinking how long a violation stays live |
| **Tier 0 catch rate** (precision/recall of high-risk-topic routing) | Effectiveness of the only real-time prevention lever in the system |
| **Tier 1 catch rate** (% of guarantee-language/price-mismatch patterns caught pre-send) | Effectiveness of the cheap deterministic layer |
| **Time-to-detect** (send → Tier 2 flag) | Whether the async loop is actually fast enough to matter |
| **Time-to-remediate** (flag → rep action / thread paused) | Whether detection translates into a stopped follow-up sequence, not just a log entry nobody acts on |
| **Judge–human agreement (Cohen's κ)** | Trustworthiness of Tier 2's scoring itself — target κ > 0.8 (see `eval-framework.md` §5) |
| **Tier 0/1 rules added per cycle from Tier 2 findings** | Whether the feedback loop is actually closing — makes the "gets smarter over time" claim measurable instead of aspirational |
| **Added p95 latency, autonomous send path (Tiers 0+1 only)** | Must stay within NFR1's tight budget — this is the one path with a real latency constraint |
| **Downstream: lead conversion rate, delta pre/post** | Ultimate business metric |

## 10. Rollout Plan

1. **Shadow mode** — all three tiers run on all traffic, but only in observe mode: Tier 0/1 log what they *would have* blocked or rerouted without actually doing so, Tier 2 scores everything. Purpose: build the calibration/gold-set and measure what each tier would have done against real traffic before any of it takes effect.
2. **Limited pilot** — turn on real Tier 0/1 enforcement (actual reroute/block) for a small number of pilot dealerships, ideally ones already flagged as higher-risk (financing-heavy stores, states with stricter advertising/lending disclosure rules). Tier 2 stays fully async throughout — it doesn't need a pilot gate since it never blocks anything.
3. **Gradual threshold tightening** — start Tier 0's classifier conservative (over-routes to human-assist), tighten as Tier 2's findings and dealer feedback validate where the real risk boundary is.
4. **General availability** — default Tier 0/1 configuration ships for all dealers, with the existing per-dealer configurability pattern extended to risk-category thresholds.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Tier 0's classifier misses a genuinely high-risk message and it slips into autonomous generation | Defense in depth, not a single point of failure — Tier 1's deterministic patterns are a second line, Tier 2's fast detection/remediation is a third; no single tier is assumed sufficient on its own |
| Judge model itself is biased or drifts over time | Calibration against a human-labeled gold set on a fixed cadence; cross-family spot-check against a frontier judge on a sample (see `eval-framework.md` §5) |
| Tier 0/1 latency budget erodes the speed advantage that's the entire product thesis | Hard latency budget (NFR1) applies only to the cheap tiers by design — no LLM judge call sits anywhere on the send path |
| Confidence-signal calibration degrades as inventory, promotions, or the underlying model change | Rolling recalibration cadence rather than a one-time calibration, since the statistical guarantee depends on an assumption (calibration/live-traffic exchangeability) that a live dealership environment will routinely violate (see `eval-framework.md` §3.4) |
| Dealers perceive the layer as the AI "getting worse" or slower | Frame and surface it as a trust feature in-product (dashboard, "AI acceptance rate" style metric dealers already understand from Service AI) rather than a hidden backend change |
| Over-blocking at Tier 0 erodes trust in the AI internally (false positives treated as the AI being unreliable) | Track and report Tier 0 precision, not just recall — a classifier that's accurate about *when* it's uncertain is different from one that's just cautious |

## 12. Open Questions

- What's the acceptable false-negative rate for Tier 0's topic classifier, and who owns setting it — Tekion centrally, or per dealer given FR1's configurability precedent?
- Tier 1's inventory-match check depends on a live lookup — in a fast-moving inventory system, is a millisecond-stale mismatch a block or just a flag? Getting this wrong in either direction has a real cost (false blocks on correct replies vs. missed stale-price sends).
- Should routing thresholds be globally tuned by Tekion or fully dealer-configurable — and if dealer-configurable, what's the default risk tolerance that ships out of the box?
- How does this interact with the "one-click disengage" feature — should a dealer disengaging AI on a lead also surface why (e.g., a Tier 2 violation flagged on that thread before the disengage)?
