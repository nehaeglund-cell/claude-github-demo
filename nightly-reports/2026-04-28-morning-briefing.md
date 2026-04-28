# 🌙 Nightly AI Report — 2026-04-28

**Generated:** 2026-04-28 02:00 UTC  
**Jobs run:** Pentest Coverage Analysis · Dependency Audit · PR Review Sweep

---

## At a Glance

| Job | Status | Key Findings |
|---|---|---|
| Pentest Coverage | ⚠️ Gaps found | 6/6 Payment API operationIds uncovered; 4 High/Medium findings untested |
| Dependency Audit | ⚠️ 3 observations | `swagger-parser` SSRF risk; `logback` on EOL branch; no critical CVEs |
| PR Review Sweep | 🔴 4 PRs need changes | 7 blocking findings across #78, #77, #76, #29 |

---

## Critical Items (Needs Attention Today)

### 1. PR #78: `PENTEST_CREATE_ISSUES=true` in PR review workflow 🚨

Adding `PENTEST_CREATE_ISSUES=true` to `claude-code-review.yml` will cause issue creation on every PR push — not just nightly runs. This will flood the issue tracker. Additionally, the Stripe test step URL is being overridden with the Payment API secret, which will silently misdirect the entire Stripe suite.

**Action:** Request changes on PR #78 before merge.

### 2. PR #76: Compilation break — `apiClient` field does not exist 🚨

`PaymentOwnershipCheckTest` references `apiClient.get(...)` but `PentestBase` exposes the field as `client`. This will break the build if merged.

**Action:** Request changes on PR #76 before merge.

### 3. PR #77: POJO + `System.out.println` in `PaymentFinding` 🚨

`PaymentFinding` should be a record, and `printSummary()` uses `System.out.println` which is invisible in CI.

**Action:** Request changes on PR #77 before merge.

### 4. PR #29: `SeverityClassifier` violates all rules it introduces 🚨

The same PR that adds `var`, immutable collections, no-println, and no-magic-numbers rules ships a class with all four violations. Dead instance fields never referenced. `System.out.println` calls in the core `classify()` method.

**Action:** Request changes on PR #29 — fix `SeverityClassifier` to comply with the rules it introduces.

---

## Security Findings — Untested in CI

All four High/Medium Payment API vulnerabilities documented in CLAUDE.md have **zero automated coverage** because `PentestSuiteTest` is disabled:

| Finding | Severity | Risk |
|---|---|---|
| IDOR — `GET /payment/status/{id}` only checks `merchantId` | **High** | Any merchant-authenticated caller can read any user's payment status |
| IDOR — `GET /payment/summary/{id}` has no `userId` check | **High** | Full payment details accessible to anyone with `merchantId` |
| Cross-user session not bound to `userId` | **Medium** | Session of user A usable by user B at the same merchant |
| Empty `sessionId` bypass on `GET /payment-types` | **Medium** | `sessionId=""` skips validation entirely |

These findings will not be auto-detected until `PentestSuiteTest` is re-enabled.

---

## Dependency Audit Recommendations

| Priority | Dependency | Action |
|---|---|---|
| Medium | `swagger-parser 2.1.21` | Verify `ParseOptions` disables remote ref resolution in CI; upgrade to 2.1.22+ |
| Low | `logback-classic 1.4.14` | Upgrade to `1.5.18` (supported branch) |
| Info | `nimbus-jose-jwt 9.37.3` | Monitor for updates; no CVEs at current version |

No CVSS ≥7 CVEs confirmed in any dependency at current versions.

---

## PR Review Verdicts

| PR | Verdict | Blocking Issues |
|---|---|---|
| #78 | REQUEST_CHANGES | Issue opt-in violated; Stripe URL broken |
| #77 | REQUEST_CHANGES | POJO not record; `System.out.println` |
| #76 | REQUEST_CHANGES | Compilation error (`apiClient`); missing `final`/private ctor |
| #29 | REQUEST_CHANGES | `SeverityClassifier` violates all new rules |
| #24 | COMMENT | `NumberFormatException` on bad config value |

---

## Recommended Actions

1. **Today:** Block merge of PRs #78, #77, #76, #29 — all have blocking findings
2. **This week:** Re-enable `PentestSuiteTest` when Payment API staging target is available — 4 High/Medium IDORs need regression coverage
3. **Next sprint:** Upgrade `logback-classic` to `1.5.x`; verify `swagger-parser` external ref resolution settings
4. **PR #24:** Can merge after addressing the `NumberFormatException` advisory (or accepting the risk)

---

## Detailed Reports

- [Pentest Coverage Analysis](./2026-04-28-pentest-coverage.md)
- [Dependency Audit](./2026-04-28-dependency-audit.md)
- [PR Review Sweep](./2026-04-28-pr-reviews.md)
