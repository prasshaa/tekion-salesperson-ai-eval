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

Insert a **Confidence-Routed Evaluation Layer** between "Salesperson AI drafts a reply" and "reply is sent." Every drafted reply is scored on two independent axes and combined into one routing decision:

1. **Quality score** — an LLM-as-judge rubric score across five weighted dimensions (grounding accuracy, completeness, escalation judgment, tone/brand fit, safety/compliance). Full rubric and judge prompt in `eval-framework.md`.
2. **Confidence score** — an objective, model-derived signal for *how sure the system should be* that the quality score is right, combining self-consistency (does the model agree with itself across repeated sampling) with a semantic-uncertainty measure. Full methodology in `eval-framework.md`.

The two scores combine into a **routing decision**:

| Combined signal | Action |
|---|---|
| High quality score + high confidence | **Auto-send** — no added latency |
| High quality score + low confidence | **Send with flag** — goes out immediately (preserves speed) but is queued for post-hoc human spot-check within the hour |
| Low quality score (any confidence) | **Hold for human review** before send — reserved for the minority of cases where the judge itself flags a likely violation (e.g., an unverifiable claim or a missed escalation) |
| Safety-dimension hard-fail (any score) | **Hard block** — never sent unreviewed, regardless of confidence; this is a deliberate override, not a threshold |

This is the same shape of decision Tekion's own **Accounts Payable AI** already makes publicly (high-confidence invoice fields auto-fill; low-confidence fields are flagged for human review) — this PRD proposes applying that same confidence-based-routing philosophy, which Tekion has already proven out on the back-office side, to the customer-facing side of the product.

## 7. Requirements

### Functional
- **FR1**: Every Salesperson AI reply is scored against the five-dimension rubric before or immediately after send (see routing table above for timing).
- **FR2**: A confidence score is computed per reply using at minimum one objective uncertainty signal (self-consistency or semantic entropy) — not verbalized confidence alone, which the literature shows is the least reliable single signal (see `eval-framework.md` §3).
- **FR3**: Any reply scoring below threshold on the Safety dimension is hard-blocked from unreviewed send, regardless of aggregate confidence.
- **FR4**: Routing thresholds are configurable per dealer (mirrors existing tone/FAQ/hours configurability) — a dealer in a highly regulated state, or one that's been burned before, should be able to set a stricter bar.
- **FR5**: Every scored reply, its rubric breakdown, confidence score, and routing decision are logged and queryable — extends the existing Communication Hub audit trail rather than replacing it.
- **FR6**: A rolling dashboard (judge/human agreement rate, safety violation rate, escalation precision, score trend by dimension) is available to the CRM AI product team and, in a lighter form, to dealers.

### Non-functional
- **NFR1 — Latency budget**: The evaluation layer must add no more than ~300–500ms to the auto-send path (async judge scoring where possible; a fast objective-signal check on the hot path, full judge scoring can complete just after send for the flag/audit case).
- **NFR2 — Cost budget**: Judge-model calls are the dominant cost driver. Given Tekion's platform already runs an **LLM Gateway** for multi-provider routing and cost tracking (per their own AI Platform Engineer job description), the judge should run on a smaller/distilled model for the hot path with a frontier-model spot-check on a sample, not a frontier judge on every message.
- **NFR3 — Multi-tenant isolation**: Consistent with how Accounts Payable AI's GL-code learning is scoped per store ("gets smarter with every correction, without affecting any other store"), any per-dealer threshold tuning or calibration data must stay isolated to that dealer.
- **NFR4 — Explainability**: Every routing decision must be traceable to specific rubric dimensions and the specific span of the reply that triggered it — not just a single opaque score.

## 8. System Design (Architecture Summary)

```
Lead message received
        │
        ▼
Salesperson AI generates draft reply
  (existing system — grounded in live inventory/deal context)
        │
        ▼
┌─────────────────────────────────────────┐
│   CONFIDENCE-ROUTED EVALUATION LAYER      │
│                                           │
│  ① Fast objective confidence check        │
│     (self-consistency sample / entropy)   │
│                                           │
│  ② LLM-as-judge rubric scoring            │
│     (5 dimensions, weighted)              │
│                                           │
│  ③ Routing decision                       │
│     auto-send / send+flag / hold / block  │
└─────────────────────────────────────────┘
        │
        ▼
   Route per decision table (§6)
        │
        ▼
Audit log + dashboard (extends Communication Hub)
```

Full technical detail — the judge prompt itself, how the confidence signal is computed, why these specific methods over alternatives — is in **[`eval-framework.md`](./eval-framework.md)**, which this PRD deliberately keeps separate so the product framing here doesn't get lost in implementation detail.

## 9. Success Metrics

| Metric | What it tells us |
|---|---|
| **Safety violation rate** (unauthorized commitments / discriminatory language / PII mishandling per 1,000 sends) | The core risk metric this whole layer exists to move |
| **Escalation precision & recall** | Is the system correctly identifying when a human is actually needed — false negatives here are the dangerous failure mode, false positives just cost speed |
| **Judge–human agreement (Cohen's κ)** | Is the automated judge actually trustworthy, or scoring drift/noise — target κ > 0.8, industry-typical threshold for a production-grade judge (see `eval-framework.md` §5) |
| **% of messages auto-sent with zero added latency** | Proxy for whether the layer is preserving the product's core speed advantage |
| **Added p95 latency on auto-send path** | Must stay within NFR1's budget |
| **Cost per evaluated message** | Judge-model spend, tracked against the Gateway's existing cost-tracking |
| **Downstream: lead conversion rate, delta pre/post** | Ultimate business metric — the hypothesis is that this layer is conversion-neutral-to-positive (catches the failures that would have cost a deal or a complaint) without being a conversion drag from added friction |

## 10. Rollout Plan

1. **Shadow mode** — layer runs on all traffic, scores and routes are logged, but nothing changes about what actually gets sent. Purpose: build the calibration/gold-set and measure what the routing *would have done* against real traffic before it affects anything.
2. **Limited pilot** — enable actual hold/block routing for a small number of pilot dealerships, ideally ones already flagged as higher-risk (financing-heavy stores, states with stricter advertising/lending disclosure rules).
3. **Gradual threshold tightening** — start conservative (more messages routed to send+flag than the steady state would need), tighten thresholds as judge-human agreement and dealer feedback validate the calibration.
4. **General availability** — default routing config ships for all dealers, with the existing per-dealer configurability pattern extended to risk thresholds.

## 11. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Judge model itself is biased or drifts over time | Calibration against a human-labeled gold set on a fixed cadence; cross-family spot-check against a frontier judge on a sample (see `eval-framework.md` §5) |
| Added latency erodes the speed advantage that's the entire product thesis | Async scoring on the hot path, hard latency budget (NFR1), fast objective-signal check before full judge scoring |
| Dealers perceive the layer as the AI "getting worse" or slower | Frame and surface it as a trust feature in-product (dashboard, "AI acceptance rate" style metric dealers already understand from Service AI) rather than a hidden backend change |
| Over-blocking erodes trust in the AI internally at Tekion (false positives treated as the AI being unreliable) | Track and report escalation precision, not just recall — a routing layer that's accurate about *when* it's uncertain is different from one that's just cautious |

## 12. Open Questions

- Does Tekion's existing LLM Gateway already expose per-call confidence/logprob data, or would this require a new capability in that layer?
- Should routing thresholds be globally tuned by Tekion or fully dealer-configurable — and if dealer-configurable, what's the default risk tolerance that ships out of the box?
- How does this interact with the "one-click disengage" feature — should a dealer disengaging AI on a lead also surface why (e.g., was the AI's confidence trending low on that thread before the disengage)?
