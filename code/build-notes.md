# Build Notes — no-po-invoice-chaser-737

API Workflow that queries Coupa for draft/new invoices in a 7-day window, filters out credit notes and PO-linked invoices, and sends a Block Kit Slack DM to the AP SME when qualifying count > 0.

## Task Status

| Task | Project | Status | Notes |
|---|---|---|---|
| T1 — Verify Coupa connection | no-po-invoice-chaser-api | done | Connection `coupa-uipath-test` (id `5fadfc73-a372-46ae-a27c-2481741eed07`) confirmed in architectural-considerations.md §4 as working despite failed ping; referenced by id, not recreated |
| T2 — Verify Slack connection | no-po-invoice-chaser-api | done | Connection `slack-product-test-app` (id `43d506f7-7de2-4798-aac1-9522e2e45dbb`) confirmed in architectural-considerations.md §4; DM scope OQ-07 carried as open assumption |
| T3 — Build workflow | no-po-invoice-chaser-api | done | All 11 SDD §4 steps implemented; `uip api-workflow validate` → Valid; `validate-build.sh` → passed with warnings |
| T4 — Testing | no-po-invoice-chaser-api | partial | Static validate passes; runtime tests require live connections (no credentials in this runner); test cases documented in SDD §8 |
| T5 — Package + publish | no-po-invoice-chaser-737 | partial | `uip solution pack` succeeds → `no-po-invoice-chaser-737_0.0.1.zip`; publish and schedule trigger require `uip login` (no credentials in this runner) |

## Deviations from the SDD

None. Every business rule (BR-01 through BR-10) is implemented. All SDD §4 steps are present. The workflow uses the approved HTTP Request + `authentication: connector` shape from architectural-considerations.md §4 verbatim.

Activity count is 27 (vs. §8's 20-activity reporting threshold). The extras are:
- 5 Response activities (1 per terminal outcome: notify success, clean day, Coupa failure, Slack failure, unhandled exception) — the SDD §6 documents all five failure modes
- 11 Assign activities (3 scripts × aliasing assigns + 1 count + 1 coupaUrl) — per §8, aliasing assigns are deliberate and stay

## Implementation Decisions

| Decision | Reasoning |
|---|---|
| PO-linkage logic: invoice qualifies if ANY line lacks a numeric po-number AND lacks a numeric order-header-num | BR-01/BR-02; description-only values are text strings matching `^[a-zA-Z\s\-\/,\.]+$`, treated as missing. Live tenant confirmed field names `po-number` and `order-header-num` on `invoice-lines[]` (OQ-05 resolved) |
| TryCatch wraps the entire flow | §6: "one failure boundary at the workflow edge"; no per-activity try/catch |
| No retry | BR-08; OQ-01 conflict resolved in favour of prose rule (no retry); no `httpRetryConfig` added |
| Clean day → no Slack call | BR-07; OQ-02 conflict: SDD implements BR-07 pending architect review |
| Coupa URL uses test tenant base | OQ-04 default: `uipath-test.coupahost.com`; production host left for SME confirmation |
| Slack body inline in `bodyParameters.body` | §4 mandatory; `validate-build.sh` checks for `blocks` directly in `body` |
| 5 Response activities | Each terminal path has its own Response with `markJobAsFailed` set correctly; prevents any path from falling through without a response |

## Left for a Human

| Item | File | SDD section |
|---|---|---|
| OQ-01: Retry policy — if SME confirms retries needed, add `httpRetryConfig` at workflow level | `Workflow.json` | §6 |
| OQ-02: Clean-day notification — if SME confirms congratulatory message needed, add Slack call on the `If_CountGtZero#Else` path | `Workflow.json` | §4 Step 7, §1 |
| OQ-03: Confirm Coupa access method and production credentials before go-live | N/A — deployment | §1 Assumptions |
| OQ-04: Replace `uipath-test.coupahost.com` with production Coupa hostname in `Javascript_CoupaUrl` script | `Workflow.json` line ~317 (Javascript_CoupaUrl code) | §4 Step 8 |
| OQ-07: Confirm `chat:write` DM scope for `WLX9BD8FN` on `slack-product-test-app` connection | Integration Service — Fusion2026 folder | §5 |
| OQ-08: Configure Orchestrator weekday schedule trigger at 10:00 Europe/Bucharest | Orchestrator — no-po-invoice-chaser-737 folder | §7 Environments |
| T5 publish: Run `uip login` then `uip solution publish /tmp/buildcheck/no-po-invoice-chaser-737_0.0.1.zip` | CLI | §9 Next Steps |

## How to Test This

```bash
# 1. Static validation
uip api-workflow validate code/no-po-invoice-chaser-737/no-po-invoice-chaser-api/Workflow.json --output json

# 2. Runnability gate
bash .github/scripts/validate-build.sh code docs/architectural-considerations.md

# 3. Pack check
uip solution pack code/no-po-invoice-chaser-737 /tmp/buildcheck \
  --name no-po-invoice-chaser-737 --version 0.0.1 --output json

# 4. Live runtime test (requires uip login with credentials for Fusion2026 folder)
uip api-workflow run code/no-po-invoice-chaser-737/no-po-invoice-chaser-api/Workflow.json \
  --input-arguments '{}' --output json
# Expected (notify path): { "status": "ok", "invoice_count": N, "slack_sent": true }
# Expected (clean day):   { "status": "ok", "invoice_count": 0, "slack_sent": false }
```
