# Part 4 – QA Leadership & AI Strategy

**Principle:** AI accelerates engineering work; humans remain accountable for quality decisions. Shared ownership: developers own coverage for the code they build; QA drives risk, cross-system coverage, regression strategy, exploratory testing, and release confidence; DevOps partners on CI/CD, environments, observability, and alerts. AI supports all three — it is not an approval authority.

With **5 BE / 4 FE / 2 QA / 1 DevOps**, QA cannot write every test. With nine developers and two QA engineers, a model where QA owns all test creation would quickly become a delivery bottleneck.

---

## 4.1 AI-First QA Operating Model

### Test ownership

| Layer | Primary owner | QA role |
|---|---|---|
| Unit / component | Developers (BE/FE for their code) | Review high-risk gaps |
| Integration / API / contracts | Backend + shared | Critical hops, Ack semantics, negatives |
| Frontend / SDK | Frontend | Host-page, reconnect, render scenarios |
| E2E (event → decision → WS → popup) | Shared; QA drives strategy | Critical-path design, exploratory |
| Regression strategy | QA drives; Dev contributes when impacting | Scope approval |
| Production quality | Shared (Dev emit, DevOps alert, QA define SLIs) | Funnel ratios, release confidence |

### Pull Request validation

```
Developer change
→ AI helps generate/update tests
→ Developer reviews generated code (assumptions, asserts, isolation)
→ Automated PR gates (fast/deterministic)
→ AI may summarize failures / suspicious diffs
→ Human code review
→ Merge only when gates pass
```

**PR gates:** unit, integration, API/contracts, targeted FE tests, lint/typecheck, and one lightweight critical-path smoke when the environment is stable. Broader regression stays off every PR. AI assists review; it does not approve its own tests.

### Automated test generation

Claude / ChatGPT / Cursor help with: cases from requirements, Playwright/API skeletons, edge-case suggestions, updating tests after refactors, test data/helpers. Engineers must verify assumptions, selectors/contracts, assertions, isolation, and that the test actually protects the requirement — especially for async WebSocket popup delivery, where REST success ≠ delivery.

### Automated test review

AI flags: sleeps, weak asserts, duplication, fragile selectors, missing negatives/cleanup, and suspicious retries. AI can also flag potential coverage gaps based on the code diff and existing tests. Findings are **recommendations**; humans accept or reject.

### Smoke

Small, fast suite after staging (and production where safe): **event → correct decision → WebSocket delivery → popup render**. Intentionally tiny — a release brake, not coverage.

### Regression

Risk-based, not “run everything.” Scope by impacted services, campaign/rule changes, critical delivery flows, and recent production defects. AI may propose impacted tests from the diff; **QA/humans approve final scope**.

### Release validation

**Automated results + known risks + production readiness + human judgment.** Green CI alone is not a release approval — especially for campaign systems where silent post-Ack loss or wrong-fire can look healthy on API metrics.

### Production monitoring

Funnel: API accepted → ingested → decided → WS send → SDK received → rendered; plus latency, errors, duplicates, WS health, customer-specific anomalies. AI helps correlate and summarize; humans set thresholds and escalate.

### Failure analysis

When a failure is detected, collect traces/logs/network/metrics; AI may summarize and correlate evidence, but engineer/QA validates the hypothesis, classifies product vs test vs environment, and drives the fix plus regression coverage. AI works from **evidence**, not a guessed root cause from one error string.

### Compact workflow

```
Requirement / Code Change
→ Developer + AI generate/update tests
→ Human review
→ PR automated quality gates
→ Merge
→ Staging smoke + risk-based regression
→ Human release decision
→ Production monitoring
→ AI-assisted failure analysis
→ Human-validated fix + regression coverage
```

---

## 4.2 Human Judgment

### Always require meaningful human review

- Release decisions and risk acceptance  
- Accepting/rejecting AI-generated tests  
- Ambiguous requirements and whether automation proves intended behavior  
- Whether a failure is acceptable to ship  
- Security/privacy-sensitive behavior  
- Changes to critical campaign / audience / decision rules  

### Greatest AI engineering risks (for this team)

1. **Confident wrong assumptions** — invented Ack meaning, selectors, APIs, or product behavior.  
2. **Passing tests that assert the wrong thing** — “some popup” instead of correct campaign/session.  
3. **Flake masking** — sleeps/retries that hide async delivery races.  
4. **Low-value coverage volume** — many tests, little risk reduction; QA becomes a review bottleneck.  
5. **Sensitive data in prompts** — customer/profile/campaign payloads leaked into Claude/ChatGPT without policy controls.

### Never fully delegated

Final release approval; risk acceptance; incident severity / root-cause calls; security/privacy decisions; whether coverage is *sufficient* for a change.

### Preventing AI-generated defects from reaching production

Human PR review; deterministic CI gates; validate the generated test (including intentional product breaks to prove it fails correctly); no AI self-approval; restrict production access; sensitive production/customer data should only be used according to the company’s approved AI/data-handling policy, and secrets and unnecessary PII should never be included in prompts; traceable Bitbucket PR history; staged rollout + funnel monitoring for delivery changes.

**Bottom line:** With two QA engineers, quality scales only if developers own their tests and AI accelerates draft work — while humans keep the judgment that green automation cannot replace.
