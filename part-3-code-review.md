# Part 3 – Code Review

## 3.1 Highest-risk issues

Reviewing this as a PR for real-time popup delivery. The main risk is not style — it is a **false green**: the test can pass without proving the campaign flow worked.

1. **No trigger / no event**
   - **Problem:** After `goto`, the test waits and looks for a popup. It never performs or verifies the user action that should fire the campaign.
   - **Risk:** Passes on a pre-existing popup, another campaign, or unrelated page content. Does not protect event → decision → WebSocket → render.
   - **Improve:** Arrange eligible state, perform the real trigger, then assert the expected popup.

2. **`waitForTimeout(5000)`**
   - **Problem:** Fixed sleep instead of waiting for an observable condition.
   - **Risk:** Flakes when delivery is slower than 5s; wastes time when it is faster. Hides races against async WebSocket delivery.
   - **Improve:** Web-first waits on popup readiness (with a timeout tied to the delivery SLA). No arbitrary sleeps.

3. **`expect(await popup.isVisible()).toBeTruthy()`**
   - **Problem:** Manual boolean read; no auto-retry. Snapshot of visibility at one moment.
   - **Risk:** Fails while the popup is still in flight; or passes on a brief flicker. Weak failure messages.
   - **Improve:** `await expect(popup).toBeVisible()` (and matching web-first asserts for content/close).

4. **Selectors (`#popup`, `.close`)**
   - **Problem:** Selector stability is unclear — `#popup` and `.close` appear to be implementation-level selectors and are not shown to be stable testing contracts. They may also collide with host-page markup.
   - **Risk:** Wrong element, or silent breakage after a redesign.
   - **Improve:** Prefer `getByRole` / `getByTestId` when stable product contracts exist. Until then, use clear placeholders — do not invent production selectors as facts.

5. **Visibility ≠ correct campaign**
   - **Problem:** Asserts only that *some* `#popup` is visible.
   - **Risk:** False positive on wrong campaign/content or a leftover modal.
   - **Improve:** Assert campaign identity / expected content for the scenario.

6. **Close is not verified**
   - **Problem:** Clicks `.close` with no follow-up assertion.
   - **Risk:** Close handler broken while the test still passes.
   - **Improve:** After close, assert the popup is hidden / detached.

7. **Isolation / setup unclear**
   - **Problem:** Hardcoded `https://demo.site`; no eligible user/campaign setup or cleanup.
   - **Risk:** Depends on shared demo state; non-repeatable; polluted by prior runs.
   - **Improve:** Deterministic fixture/helper for eligible user + known campaign; unique data where possible; cleanup when hooks exist.

---

## 3.2 Rewrite

### Assumptions (not provided by the assignment)

- Host page URL and trigger control exist for tests (`BASE_URL`, `data-testid`s below are **placeholders**).
- A helper can prepare a known eligible user + campaign for this run (`setupEligibleCampaignFixture`).
- The browser/SDK session must be bound to that same test user (placeholder: `initializeTestUser(page, user)`).
- Popup exposes stable test IDs for root, campaign identity, and close.
- Delivery is async (WebSocket); we wait on the popup becoming visible, not on a sleep.
- `POPUP_SLA_MS` comes from the agreed product delivery SLA (not invented here).
- Exact selectors/APIs are unknown — marked as placeholders, not invented product facts.

```ts
import { test, expect } from '@playwright/test';
// Placeholders: replace with real fixture/helpers when available
import { setupEligibleCampaignFixture, initializeTestUser } from '../support/campaignFixtures';

// Must be set from the agreed product delivery SLA — do not invent a numeric timeout
const POPUP_SLA_MS = Number(process.env.POPUP_SLA_MS);

test('Should show expected campaign popup after trigger and close it', async ({ page }) => {
  const { campaign, user } = await setupEligibleCampaignFixture(); // unique eligible user + known campaign

  await page.goto(process.env.BASE_URL ?? '<HOST_PAGE_URL>');
  await initializeTestUser(page, user); // bind SDK/browser session to the same test user

  // Optional: only if a real readiness signal exists
  // await expect(page.getByTestId('<sdk-ready-testid>')).toBeVisible();

  // Act — business trigger (placeholder locator / action)
  await page.getByTestId('<trigger-action-testid>').click();
  // If the product triggers via SDK API instead of UI, call that helper here instead.

  const popup = page.getByTestId('<campaign-popup-testid>');

  await expect(popup).toBeVisible({ timeout: POPUP_SLA_MS });
  await expect(popup.getByTestId('<campaign-id-testid>')).toHaveText(campaign.id);
  // Or assert known title/copy from the fixture if that is the product identity signal:
  // await expect(popup.getByTestId('<campaign-title-testid>')).toHaveText(campaign.title);

  await popup.getByTestId('<popup-close-testid>').click();
  await expect(popup).toBeHidden();
});
```

---

## 3.3 Improvements

- **Web-first assertions** retry until the condition holds or the timeout expires. `isVisible() + toBeTruthy()` does not — it freezes one moment and is a common source of flakes on async delivery.
- **Sync on observables, not time.** Waiting for the expected popup (and identity) matches the real path: event → decision → WS → render. A 5s sleep does not.
- **Popup identity** prevents “any modal appeared” false greens — critical when multiple campaigns or leftover UI can show.
- **Asserting close** turns a click into a verified outcome; without it, broken dismiss behavior ships unnoticed.
- **Deterministic setup** matters for campaign systems: eligibility, frequency caps, and prior deliveries make shared demo state unreliable. Unique fixture + clear placeholders keep the rewrite honest about what we do not know yet.

---

## 3.4 AI Validation

Before approving an AI rewrite of this PR:

1. **Manual review** against the requirement: trigger → correct popup → close verified.
2. **Check for invented details** — reject hard-coded fake selectors/APIs presented as real; placeholders and stated assumptions are fine.
3. **Run locally** against a known eligible fixture.
4. **Break the behavior on purpose** (wrong campaign content, disable close, block WS/delivery) and confirm the test **fails for the expected reason**.
5. **Repeat locally / in CI** enough times to surface obvious flakes — not as proof of zero flakiness.
6. On failure, inspect **trace, screenshot, and network/WS evidence**.
7. Review **isolation/cleanup** so the test does not depend on leftover popups or shared users.
8. Confirm assertions prove **business behavior**, not merely that `#popup` exists.

**Principle:** A green run is not enough. I also need to prove the test fails when the behavior it protects is broken. AI can draft the fix; approval stays an engineering decision.
