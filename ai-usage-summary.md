# AI Usage Summary

**Tool used:** Cursor (throughout Parts 1–5).

I used Cursor to accelerate drafting and iteration — not to replace engineering judgment. For each part I reviewed AI output against the architecture diagram, the assignment requirements, and the critical delivery chain: **event received → correct decision → correct real-time delivery**.

## What I asked AI to do

| Part | How AI helped |
|---|---|
| **1** | Structure the system explanation, brainstorm architectural risks, draft the risk-based QA strategy and hotfix prioritization |
| **2** | Draft the Playwright prompt and flaky-output review for async WebSocket popup delivery |
| **3** | Structure the PR review of the AI-generated Playwright test and a production-quality rewrite shape |
| **4** | Design the AI-first QA operating model, ownership table, and compact workflow |
| **5** | Structure the funnel-based incident investigation and validation/prevention sections |

## What I accepted

- Framing REST and WebSocket as separate channels (REST success ≠ popup delivered)
- Risk focus on silent post-Ack loss, wrong-fire, and session-bound push delivery
- Shared quality ownership (developers own coverage of the code they build; QA drives risk/E2E/regression)
- High-value Playwright scenarios over broad reconnect matrices
- PR review priorities: missing trigger, fixed sleeps, weak asserts, false greens
- Funnel isolation for incidents (affected vs control; find the first diverging hop)
- Principle that AI assists, humans remain accountable for quality and release decisions

## What I modified or rejected

- **Unsupported architecture as fact** — Ack semantics, whether ingestion runs after the REST response, frequency caps/dedup location, and WebSocket session mapping were treated as questions/assumptions, not confirmed design
- **Invented details** — selectors, APIs, test hooks, numeric SLAs, and undrawn infrastructure (queues/offsets) presented as facts
- **Generic answers** — testing-pyramid language, “API + UI + database” hotfix priorities, and generic incident checklists not tied to this funnel
- **Over-broad automation** — large reconnect matrices and full E2E on every PR
- **Flake masking** — fixed waits/retries used to hide async delivery races
- **Green = ship** — treating automation pass as sufficient release confidence
- **Overconfident / overly punchy wording** — revised for clearer, more professional engineering tone
- **AI self-approval** — AI may recommend; it does not approve its own tests or release decisions

## How I validated the final result

- Re-read the architecture diagram and assignment requirements for each part
- Distinguished known facts from assumptions throughout
- Checked each answer against its specific required sections and word limits (Parts 1, 2.2, 5.2)
- Reviewed generated Playwright guidance/code using normal engineering judgment (placeholders where product contracts are unknown; prove tests fail when behavior breaks)
- Ensured Parts 1–5 stay consistent on the critical flow and on shared ownership / human accountability
- Did **not** claim execution against a live Solitics environment or that repeated green runs prove zero flakiness

**Bottom line:** AI accelerated structuring and drafting. Final technical decisions, risk prioritization, and submission quality remain my responsibility as QA Manager.
