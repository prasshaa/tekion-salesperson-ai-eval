# Confidence-Routed Evaluation Layer for Tekion Salesperson AI

**A case study for the Tekion AI Product Manager role — proposing and prototyping an evaluation and confidence-routing layer for Salesperson AI, Tekion's CRM AI agent.**

Author: [Prathyusha Vedula](https://linkedin.com/in/prathyushavedula) · [prasshaa](https://github.com/prasshaa)

---

## Why this exists

Tekion has shipped a real, live agentic AI portfolio — T1, Accounting AI, CRM AI, Service AI — and is one of the few automotive platforms with an [ISO/IEC 42001](https://tekion.com/resources/iso-42001-tekion-responsible-ai) AI governance certification. That's a genuinely strong foundation.

This project asks one narrow, concrete question about one of those agents — **Salesperson AI**, which responds to and follows up with sales leads autonomously, 24/7 — and answers it end to end, the way I'd approach it as a PM:

> *When Salesperson AI sends a message to a real customer with no human in the loop, what decides whether that message goes out — and how would you build, evaluate, and prove out that decision layer?*

Everything here — the PRD, the eval methodology, the working prototype — is built from public information about Tekion's actual product plus my own applied background running LLM-as-judge evaluation pipelines in production (I currently build offline/online evals and guardrails for an autonomous medical-coding agent at Arintra). Numbers in the demo are illustrative, not real Tekion data — the point is to show the way I'd think and build, not to claim insider knowledge.

## What's in this repo

| File | What it is |
|---|---|
| [`PRD.md`](./PRD.md) | Full product requirements doc — what Salesperson AI is today, the problem, the proposed solution, requirements, success metrics, rollout plan |
| [`eval-framework.md`](./eval-framework.md) | The deep technical spec — LLM-as-judge methodology, confidence-scoring approaches, the actual judge prompt, routing logic, calibration plan, and how each choice maps to current industry practice |
| [`demo/index.html`](./demo/index.html) | A working, interactive prototype — a scored rubric applied to sample Salesperson AI conversations, plus an aggregate quality dashboard |

## Live demo

Once hosted on GitHub Pages: `https://prasshaa.github.io/tekion-salesperson-ai-eval/demo/`

(Push instructions in the last section below.)

## TL;DR of the proposal

Salesperson AI is already one of the more autonomous agents in Tekion's portfolio — it sends real messages to real leads with no human in the loop for the routine case. That's the right design for speed, but it means the actual product risk isn't "does the agent answer well" — it's **"what happens the one time it doesn't."** A single unauthorized promise (a financing guarantee, an incorrect price lock) sent to a real customer is a materially different failure than a low-quality chatbot answer.

The proposal: a **confidence-routed evaluation layer** sitting between response generation and send — combining an **LLM-as-judge rubric score** (is this response good) with an **objective confidence signal** (how sure is the model it's right) into one routing decision: auto-send, hold for human review, or escalate. Full detail in `PRD.md` and `eval-framework.md`.

---

*This is an independent case study built for interview purposes. Not affiliated with or endorsed by Tekion Corp. All product names, screenshots, and public statements referenced belong to Tekion and are cited from their public website and job postings as of August 2026.*
