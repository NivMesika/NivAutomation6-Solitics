# Part 1 – System Understanding & Risk-Based QA Strategy

## 1.1 Explain the System

This is not an API that returns a popup. The SDK posts the visitor’s action over HTTPS and, separately, holds a WebSocket so we can push a render instruction later. If those channels disagree about who the user is, we miss - or we show the offer to the wrong person.

**Client Browser / Customer Website** - Host page we do not control. Actions start here; the popup is painted on someone else’s DOM.

**SDK** - Init, collect, REST, WSS, render. If the SDK fails, a correct backend still means a missed campaign.

**REST API Layer** - Synchronous front door. HTTPS in, Ack/Config/Data out. Ack is undefined. I would clarify: received, durably ingested, or fully processed?

**Event Ingestion & Processing** - Parse, enrich, forward. Drawn below the API Layer, not proven to run after the REST response. If it does, a 200 can succeed while the event never reaches the engine.

**User Profile & Segmentation** - Who this is, which audiences. Stale membership is a silent wrong-decision.

**Real-time / WebSocket Server** - Persistent WSS. Push path, not the event path. Session mapping is not shown. I assume it exists, or Action Sender has no reliable target.

**Campaign Manager** - Authoring. Off the hot path.

**Audience & Rules** - Conditions. Frequency caps and dedup are not shown. I would ask where they live - here, in the Decision Engine, or nowhere. If nowhere, retries can double-fire.

**Decision Engine** - Given this event and this profile, fire or not, and which campaign.

**Action Sender** - Turns a yes into a WS payload. If it cannot bind a live session, a correct decision never reaches the user.

**Campaigns DB** - live config. **User Profiles DB** - attributes and segments. **Analytics DB** - after-the-fact counts; it does not cause a popup.

**Flow:** action → SDK → REST POST/response (sync) → ingestion → profile/segment → rules + decision → Action Sender → WS message → SDK render.

REST is request/response. REST and WSS are separate channels and can become temporarily unsynchronized. Ingestion timing versus the REST response is unspecified. Two-way DB arrows hide whether we evaluate on an updated profile or yesterday’s segment.

1. **Ack then drop (API → Ingestion).** If Ack is not durable processing, an event can look successful and still be lost while API latency looks normal. High risk because the failure is silent.

2. **Wrong fire (Profile + Decision Engine).** Wrong/stale segment, wrong campaign, or duplicate. Missed revenue or the wrong offer. High risk because a popup appearing keeps infrastructure green.

3. **Push / session bind (Action Sender → WS → SDK).** A yes never reaches this tab, hits a stale or foreign socket, or the SDK cannot render. Users see nothing, or the wrong user sees the offer. High risk: only customer-visible hop, and session mapping is not shown.

## 1.2 QA Strategy

This system fails in hops, not in a single box. REST health is not delivery health. Effort goes to proving three things: the event actually entered processing, the decision was right for that user, and the instruction landed on the right live session and rendered.

Quality is shared. Developers own unit/component coverage for the code they build, and contribute to integration and E2E when their changes affect those flows. QA drives risk analysis, critical scenarios, and E2E/regression strategy.

**Unit** - What: rule predicates, segment calculation, SDK payloads, idempotency / deduplication logic. Why: cheapest catch of business-critical decision logic. Who: Developers. QA reviews high-risk rule coverage.

**Integration** - What: API → ingestion → profile read → decision; Action Sender → WS server; profile read-after-write. Why: where accepted events vanish and where we decide on stale data. Who: Shared. QA names the hops; developers write the tests.

**API** - What: contracts, validation, auth, Ack shape, retry/idempotency of event POST. Why: the SDK trusts Ack. Until its meaning is confirmed, we must not treat it as “campaign sent.” Who: Shared. Developers own contract tests; QA drives Ack semantics, negatives, and duplicate POST.

**SDK / Browser** - What: init, collect, REST send, WS lifecycle, popup render, reconnect, real customer host pages / different DOM environments. Why: uncontrolled DOM, ad blockers, SPA navigation. Who: Shared. Frontend owns SDK internals; QA drives host-page and reconnect.

**WebSocket** - What: connect, reconnect, payload schema, bind to the correct session, no send on unknown session. Why: this is delivery. Mis-routing is a privacy incident. Who: Shared. Backend owns protocol tests; QA drives session-routing and reconnect.

**UI** - What: popup visible, close, does not break the host page. Why: an instruction that never paints is a failed campaign. Who: Shared. Frontend owns render; QA samples on a host-like page.

**End-to-End** - What: user action → the right popup for the right user, and no fire for the wrong one. Why: only place REST, processing, decision, WS, and render meet. Who: Shared. QA drives the critical-path strategy; developers contribute when they touch the flow.

**Smoke** - What: init, one event, popup appears. Why: release brake, not coverage. Who: Shared. QA defines the path; DevOps wires it; developers keep it green.

**Regression** - What: live trigger matrix (campaign × segment × event), duplicate event, reconnect, should-not-fire. Why: new campaigns change behaviour without a code change. Who: Shared. QA drives the matrix; developers add a case for a new rule type.

**Performance** - What: ingest lag, decision time, event-to-WS-send, WS fanout, traffic spike. Why: real-time is an SLA. Who: Shared. QA defines the latency budget and spike shape; Backend and DevOps run it.

**Production monitoring** - What: funnel at each hop - REST accepted, ingested, profile resolved, decided, WS send, SDK received, SDK rendered - plus duplicate rate, WS churn, latency. Why: tests will not catch “only some customers” or a silent drop after a successful REST call. Who: Shared. Developers emit hop events; DevOps alerts; QA defines the SLIs.

**Every Pull Request** - Fast unit, API, integration, and WebSocket tests: decision/segmentation logic, SDK payloads, idempotency, API contract plus duplicate POST, ingest → decision, and WS routing against a fake session registry (unknown session is a recorded miss). If the environment is stable enough, also run one lightweight critical-path E2E smoke: event → correct decision → popup displayed. Broader browser/E2E regression stays for pre-release.

**Before Release** - Full E2E: action → correct popup on a host-like page. Must-pass: reconnect then event; duplicate REST does not double-fire; out-of-audience user sees nothing. Smoke. Regression of the live trigger matrix, not Campaign Manager CRUD. Staging p95 event-to-WS-send against the budget. Skip full browser matrix and Analytics completeness - they do not gate delivery.

**Continuously in Production** - Alert on ratio breaks (accepted much greater than ingested, fire-decided much greater than rendered), duplicate-delivery, WS churn, event-to-popup latency. Sample that connection identity matches the event’s identity. Analytics DB is a lagging indicator, not the primary health signal.

Assumption: hop counters can be emitted. If they cannot, that instrumentation is a prerequisite, not a later improvement.

## 1.3 Hotfix Release Prioritization

I would spend the three slots on the live delivery chain: event received → correct decision → correct real-time delivery.

**1. Event intake integrity (REST response → actually ingested)**
Validate: a known event gets a successful REST response and then appears in downstream processing (hop counters if available, or correlated processing logs) — not HTTP 200 alone. A duplicate POST is not processed twice.
Risk reduced: silent drop after a successful API call while campaigns stop firing.
Why this over Campaign Manager: production is taking traffic; I need to know events are not disappearing after the response.

**2. Decision correctness for a known user (profile + rules + engine)**
Validate: one in-audience user fires the expected campaign; one out-of-audience user does not; a second identical event does not fire again if the campaign is single-show.
Risk reduced: wrong offer, missed offer, or duplicate popups while infrastructure looks healthy.
Why this over Analytics or full segmentation: I can re-prove live campaigns, not rebuild the audience model.

**3. Real-time delivery to the originating session (Action Sender → WS → SDK render)**
Validate: the fire instruction arrives on the same browser session that sent the event; after a forced disconnect/reconnect it still arrives once; the popup is visible.
Risk reduced: correct decision with no popup, duplicate popup on reconnect, or a message on the wrong connection.
Why this over UI, Campaigns DB, or load tests: a broken hop wastes every correct decision. Performance testing matters, but in a hotfix window I prioritize the critical delivery path and run performance tests in a controlled environment.

Skipped: Campaign Manager, Analytics, broad UI, performance, browser matrix. They matter once we are stable. They are not the minimum chain: event received → correct decision → correct real-time delivery.
