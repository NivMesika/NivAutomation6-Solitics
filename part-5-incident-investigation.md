# Part 5 – Production Incident Investigation

**Facts (given):** Popup delivery 98% → 67% yesterday afternoon; no deployments; infrastructure healthy; API latency normal; no DB alerts; **only some customers** affected.

**Working model (from Part 1):** event → ingestion/processing → profile/segmentation → decision → Action Sender → WebSocket → SDK render.  
**Not assumed as fact:** queues/offsets, feature flags, or undrawn services — those are checks only if evidence points there.

**Initial hypotheses (explain subset impact + green infra):** customer-specific campaign/config change; audience/segmentation or profile freshness; ingestion/processing lag that does not show in API latency; WS/session bind failure for some identities; SDK/browser or customer-site change; campaign schedule/rules/expiry; identity mismatch across REST vs WSS. None confirmed yet.

---

## 5.1 Investigation (9 steps)

1. **Confirm blast radius and clock**  
   **Check:** Delivery % over time yesterday; first inflection point; list of affected vs unaffected customers; whether impact is still growing.  
   **Why first:** Establishes when it started and whether this is still live.  
   **Next:** If impact is rising or wrong-user delivery appears → escalate early. Else proceed with affected vs control comparison.

2. **Slice metrics: affected vs healthy**  
   **Check:** Break delivery by customer, campaign, region/env (if relevant), browser/SDK version (if telemetry exists).  
   **Why:** Facts say subset impact — find the shared attribute of the failing cohort.  
   **Next:** Strong customer/campaign correlation → prefer config/rules path. SDK/browser correlation → prefer client path. No shared attribute → keep funnel isolation open.

3. **Build a control pair**  
   **Check:** Pick 2–3 affected and 2–3 unaffected customers with comparable traffic; note customer/user/session/campaign/event IDs for correlation.  
   **Why:** Same time window, different outcome — isolates the first diverging hop.  
   **Next:** Use these IDs through every funnel stage.

4. **Funnel hop 1–2: REST accepted vs downstream processing**  
   **Check:** Compare accepted events with evidence of downstream processing, using hop counters if available or correlated processing logs otherwise. Look for drop after the REST layer without API latency change.  
   **Why:** Part 1 risk — silent post-Ack loss can look like healthy API. Do not assume a specific queue/offset architecture.  
   **Next:** Divergence here → Backend on ingestion/processing. If both cohorts match through processing → go deeper.

5. **Funnel hop 3–4: profile/segment → decision**  
   **Check:** Profile resolve success, segment membership, campaign/rule evaluation logs for the same events; compare fire/no-fire rates and which campaign was chosen. Review campaign/config **audit history** around the inflection time (Marketing/Product may have changed rules without a deploy).  
   **Why:** Config/audience changes need no deployment and often hit only some customers.  
   **Next:** Wrong/no-fire only for affected → Backend + Product/Marketing on rules/segments. Equal decisions → push path.

6. **Funnel hop 5: Action Sender → WebSocket send (including identity/session bind)**  
   **Check:** “Decision=fire” vs “WS payload sent”; whether the send targeted the correct user/session/socket (REST identity matches WS session); unknown/stale/unbound sockets; mis-routed delivery.  
   **Why:** Correct decision with no send — or send to the wrong session — explains delivery drop (or privacy impact) while API/DB stay green.  
   **Next:** Wrong-user / mis-routed delivery → escalate immediately (privacy); Backend owns bind/mapping. Send failures or bind failures → Backend (Action Sender / realtime). Correct send → client/render.

7. **Funnel hop 6–7: SDK received → popup rendered**  
   **Check:** Client telemetry: instruction received, render success/errors, SDK version; any customer-website CSP/ad-blocker/host-page change signal if available.  
   **Why:** Subset customers may share SDK version or host integration.  
   **Next:** Receive OK / render fail or never receive → Frontend/SDK (+ customer if host change). Involves Frontend only with this evidence.

8. **Form a single primary hypothesis and validate**  
   **Check:** State the first hop where cohorts diverge; pull matching logs (API/event, ingest, profile/decision, Action Sender, WS session, SDK render) for one failing event end-to-end. Involve **only** the team owning that hop (Backend / Frontend / DevOps for telemetry gaps / Product-Marketing for config).  
   **Why:** Avoid all-hands without evidence.  
   **Next:** If hop still unclear after a short timebox → escalate for more capacity/observability.

9. **Close the loop / escalate if needed**  
   Confirm the primary hypothesis with control-pair evidence, or escalate with the control-pair IDs, inflection time, and first diverging funnel hop — not a vague “delivery is down.”

**Escalate when:** impact is continuing or increasing; wrong-user or privacy-related delivery; unable to isolate the failing stage within a reasonable timebox; business-critical campaigns are affected; evidence of broader systemic failure.

---

## 5.2 Validation & Prevention

### Confirm the root cause

Reproduce or demonstrate with correlated IDs: same event path fails for an affected customer and succeeds for a control customer. Show the **first hop** where they diverge (e.g. decided fire but WS not sent; or WS sent but SDK never rendered). Treat time correlation alone (e.g. “Marketing edited a campaign”) as insufficient until that hop evidence matches.

### Validate the fix

1. Reproduce the failing control-pair path before the fix.  
2. Apply the fix in a controlled environment (staging or limited canary).  
3. Verify the specific broken hop recovers.  
4. Run critical E2E: event → correct decision → WS → correct popup.  
5. Retest affected and unaffected customer scenarios.  
6. Check adjacent risks (duplicates, wrong campaign, reconnect).  
7. Roll out and watch funnel ratios for both cohorts.

### Historical impact

From inflection timestamp + audit/analytics/logs: which customers/campaigns; how many missed vs wrong vs duplicate deliveries; whether any privacy/mis-route cases occurred. Prefer event/campaign/session-level logs over aggregate “delivery %” alone.

### Prevent recurrence

Derive controls from the **actual** failing hop, for example:

- Missing hop monitoring / customer-level alerts at that stage  
- Regression covering the failing condition (rules, session bind, SDK version)  
- Config change validation + audit trail if the cause was campaign/rules  
- Safer change control / staged rollout for delivery-affecting config  
- SDK compatibility checks if client-version specific  
- Update the runbook/postmortem with the funnel isolation steps used here

Do not add every control by default — match prevention to the root cause.
