# Part 2 – AI-Assisted Test Engineering

## 2.1 Initial Prompt to the AI Coding Assistant

```text
You are helping me write Playwright Test + TypeScript E2E tests for Solitics real-time campaign popup delivery.

## System context

This is a distributed real-time campaign system, not a simple “API returns a popup” flow.

Critical path:
user action on customer website
→ Solitics SDK collects/sends event over REST/HTTPS
→ Backend ingests/processes the event, resolves profile/segment, evaluates audience rules
→ Decision Engine decides whether to fire a campaign
→ Action Sender pushes a popup instruction over WebSocket (WSS)
→ SDK renders the popup on the host page

Important constraints:
- REST success alone does NOT prove the popup was delivered.
- REST and WebSocket are separate channels and can be temporarily out of sync.
- The E2E outcome that matters is: the correct popup rendered for the correct session/user after a valid decision.
- Do not invent architecture, selectors, endpoints, event names, campaign IDs, or test hooks. If something is unknown, use clear placeholders (e.g. `<POPUP_TEST_ID>`, `<TRIGGER_ACTION>`, `<EVENT_NAME>`) and list what you still need from me before finalizing.

## Goal

Generate a small, high-value initial suite for popup delivery. Prefer a few reliable scenarios over broad coverage.

## Scenarios to implement (priority order)

1. Eligible user performs the trigger → expected popup appears.
2. Non-eligible user performs the same trigger → popup does not appear within a defined business-relevant wait window.
3. When popup appears, campaign/content identity is correct (title/copy/campaign identifier — use placeholders if unknown).
4. Duplicate or repeated trigger for a single-show campaign → popup appears only once (state this as an assumption if single-fire policy is unknown; ask me to confirm).
5. User can close the popup successfully, and it is no longer visible.

Optional (only if I provide a reliable WebSocket readiness / reconnect hook):
6. After reconnect, a subsequent eligible trigger still delivers exactly one popup.
Do NOT expand into a large reconnect matrix in this first suite.

## Reliability expectations (hard requirements)

- No `waitForTimeout()` / arbitrary sleeps.
- Use Playwright auto-waiting and web-first assertions (`expect(locator).toBeVisible()`, etc.).
- Synchronize on meaningful conditions before asserting the popup, for example:
  - page ready / SDK initialized
  - WebSocket connected (only if a testable signal exists)
  - trigger action completed
  - then popup ready/visible
- Avoid races between sending the event and waiting for the async WebSocket-driven popup.
- Tests must be isolated and repeatable.
- Do not depend on existing production/user data.
- Use unique or deterministic test identities per run (unique user/session/campaign fixture where applicable).
- Clean up created data/configuration after the test when APIs/hooks exist.
- Do not retry actions or wrap flaky steps in loops to hide instability.
- Failures should represent real product issues or missing synchronization, not timing noise.

## Assertion strategy

Assert business behavior, not “an element exists somewhere.”

For positive cases:
- Correct popup is visible.
- Correct campaign/content is displayed.
- Where possible, assert it belongs to the expected user/session (e.g. via known test campaign content or a session-scoped identifier — do not invent one).

For negative cases:
- Assert the popup does not appear within a wait window taken from the product’s delivery SLA (ask me for the SLA if unknown; use a named constant placeholder like `NO_POPUP_WINDOW_MS`). If the window is shorter than the SLA, a negative test can pass simply because we stopped waiting too early.

Supporting evidence (optional, secondary):
- You may observe network (REST event request/response) or WebSocket frames if helpful for debugging.
- The primary E2E pass/fail criterion remains the user-visible popup outcome.

## Coding standards

- Playwright Test + TypeScript.
- Clear test names describing observable outcomes (e.g. “Should show expected popup for eligible user after trigger”).
- Prefer thin specs + reusable fixtures/helpers (and page-object style flows/validation if scaffolding already exists).
- Resilient locators: prefer `getByRole`, `getByTestId`, accessible names. Avoid brittle CSS/XPath unless no better option exists, and comment why.
- Minimal duplication; readable Arrange → Act → Assert.
- Comments only where they explain non-obvious sync or business rules.
- Keep tests independent; parallel-safe by default unless shared backend state forces serial execution (call that out explicitly).

## Missing information

First list the missing information. Where reasonable, continue with clearly marked placeholders instead of inventing implementation details. Ask me for:
1. Demo/host page URL and how the SDK is loaded/initialized in test.
2. How to perform the trigger action in the UI (or SDK API for sending the event).
3. Event name/payload contract.
4. Popup DOM contract: roles, `data-testid`s, close control, campaign identity fields.
5. How to set up eligible vs non-eligible users/segments and which campaign should fire.
6. Whether duplicate events are expected to produce one popup or can legitimately produce multiple.
7. Event → popup SLA / acceptable wait window for positive and negative asserts.
8. Any WebSocket readiness signal, reconnect hook, or test-only debug API.
9. Setup/cleanup APIs for campaigns, profiles, and delivery state.
10. Existing fixture/POM conventions in this repo, if any.

## Output format

1. First list any missing information as clarifying questions.
2. Where reasonable, continue with clearly marked placeholders instead of inventing implementation details.
3. Briefly explain how each test avoids racing the async WebSocket path.
```

---

## 2.2 Review the Flaky AI Output

The first AI draft is a starting point, not a finished suite. Flakiness here usually comes from treating async WebSocket delivery like a synchronous UI click.

### Likely causes

Most likely:

1. **Fixed sleeps / timing assumptions** — `waitForTimeout(5000)` while event→decision→WS→render latency varies by environment.
2. **Racing the WebSocket path** — asserting the popup immediately after the REST trigger, before the instruction arrives.
3. **Asserting too early** — checking visibility before SDK init or before the socket is usable.
4. **Weak selectors** — brittle CSS (`#popup`) that matches the wrong node, a host-page element, or nothing after markup changes.
5. **Shared users/campaigns** — parallel or previous runs leave eligibility, frequency-cap, or “already shown” state dirty.
6. **Duplicate / stale campaign state** — leftover popup from a prior trigger, or a second event that re-fires.
7. **Environment latency** — CI slower than local; positives timeout, negatives pass by accident.

Reconnect timing can flake too, but I would not put a large reconnect matrix in the first suite.

### How I would improve the prompt

I would tighten the first prompt with:

- Explicit sync points: page ready → SDK ready → (optional) WS connected → trigger → wait for popup ready, not a sleep.
- A named positive timeout and a named negative “no popup” window taken from the product’s delivery SLA — otherwise a negative test can pass simply because we stopped waiting too early.
- Hard ban on retries used to mask flakes.
- Unique per-test identity + required cleanup hooks.
- First list missing information; where reasonable, continue with clearly marked placeholders instead of inventing details.
- Single-fire policy stated as an assumption to confirm.

### Additional context I would give AI

Before regenerating:

- Host page URL and how the SDK initializes
- Trigger action and event name/payload
- Popup `data-testid` / role / close control / campaign identity fields
- Eligibility setup for in-audience vs out-of-audience users
- Whether one event may produce multiple popups
- Event→popup SLA
- Setup/cleanup APIs for campaign, profile, and delivery state
- Any WS readiness or test-only hooks

Without this context, AI is likely to make assumptions that can produce incorrect or flaky tests.

### Iterative AI workflow

I do not accept the first output blindly:

**generate → human review → run → inspect failure / trace / screenshot → give AI the concrete evidence → refine → re-run repeatedly**

AI can suggest locator or wait fixes. I decide whether the failure is a product bug, a bad assertion, or missing sync. If a “fix” only passes after sleeps or retries, it is not ready.

### Production readiness

I would approve when:

- Tests pass repeatedly locally and in CI (enough runs to expose obvious flakes; not a claim of zero flakiness)
- Assertions prove business outcomes, not just element presence
- No arbitrary waits
- Isolation and cleanup are real
- Failures leave useful trace/screenshot/logs
- Failures usually mean real product issues, not timing noise
- A human has reviewed the PR

AI can accelerate test creation. It does not decide whether the generated test is correct, reliable, or valuable. That remains an engineering responsibility.
