# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-22 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, UiPath Cartographer, 16 Sep 2026) |

## 1. Document Control

| Item | Value |
|---|---|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-737 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-737 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser

| Item | Value |
|---|---|
| Process Full Name | NoPoInvoiceChaser |
| Business objective | Replace the daily manual Coupa invoice review with a fully automated weekday run that identifies invoices lacking a properly linked PO and notifies the AP responsible on Slack, enforcing the no-PO-no-pay policy consistently |
| Owning department | Accounts Payable (Finance) |

**Delivery Team**

| Role | Name / Contact |
|---|---|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|---|---|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable — invoice compliance monitoring |
| Short description | Automated weekday run queries Coupa for draft/new invoices from the past seven days, excludes credit notes, counts those with no properly linked PO, and sends one Block Kit Slack message to the AP SME; sends nothing on a clean day |
| Required roles | Automation (unattended); SME receives output |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | ~194 invoices evaluated per run [SME REVIEW — confirm peak]; qualifying (no-PO) subset varies |
| Average handling time | Manual: ~daily ad hoc (timing inconsistent); Automated target: minutes per run |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — data is structured, rules deterministic; credit-note and description-only-PO cases are expected edge cases |
| Input data | Coupa invoice list: invoice date, status, invoice type, PO-linkage field, invoice ID |
| Output data | Slack Block Kit direct message to SME with qualifying count and filtered Coupa URL; no output on a clean day |

## 4. Scope

**In scope**
- Weekday execution triggered at 10:00 Romania time
- Coupa invoice retrieval via read-only query
- Date-window filtering: invoice date within the past seven days
- Status filtering: draft or new only
- Credit-note exclusion
- PO-linkage validation: proper link required, description-only PO text treated as missing
- Count of qualifying (no-PO) invoices
- One Slack Block Kit direct message to the SME including count and filtered Coupa URL
- No-message behaviour on a clean day (zero qualifying invoices)
- Clean-day congratulatory message on Slack [SME REVIEW — section 7 says send congrats; BR-007 says send nothing; confirm which applies]

**Out of scope**
- Purchase-order creation or modification
- Invoice approval or payment release
- Any Coupa record modification
- Supplier communication
- Requester direct notification or follow-up tracking
- Full invoice lifecycle automation
- Audit-retention design
- Listing individual invoices in the Slack message
- Retry, fallback and error-recovery behaviour (per BR-008; diagram shows retry — see OQ-01)

## 5. To-Be Process (High Level)

The automation is a short linear unattended process: **trigger → query → filter → count → notify**. No human review is retained in the normal path.

- The UiPath Orchestrator weekday schedule fires at 10:00 Romania time.
- The robot queries Coupa (read-only) for invoices dated within the past seven days with status draft or new, excluding credit notes.
- Each invoice's PO-linkage field is evaluated; a PO number present only in the description does not satisfy the requirement.
- The qualifying count is assembled. If zero, nothing is sent (clean-day path).
- One Slack Block Kit direct message is composed from the template placeholders and delivered to the SME by Slack member ID.
- The run ends; no Coupa records are altered.

**Steps eliminated from the manual process:** manual Coupa list filtering, manual copy of invoice fields, manual grouping by requester, manual Slack message composition and timing.

## 6. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|---|---|---|---|---|
| 1.1 | Orchestrator weekday schedule fires at 10:00 Romania time (EET/EEST) | UiPath Orchestrator | Process job starts | Trigger is schedule-based; timezone must be set to Romania (EET UTC+2 / EEST UTC+3) |
| 1.2 | Calculate date window: window_end = today's date; window_start = today minus 7 days | Automation | window_start and window_end variables set | Used for both the Coupa query filter and the message placeholders; BR-04 |
| 2.1 | Query Coupa invoice list filtered to invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Raw invoice list returned | Read-only; interface type [SME REVIEW — API or web UI]; BR-04 |
| 2.2 | Check Coupa response is valid (non-error, parseable) | Coupa | Boolean: response valid / invalid | If invalid → step 2.3; if valid → step 2.4; see image2 diagram |
| 2.3 | [DECISION] Coupa response invalid — mark run as failed, end process; no Slack notification sent | Automation | Run logged as failed | Per BR-08, BR-09; no retry in prose scope — see OQ-01 |
| 2.4 | Exclude records where invoice type = credit note | Automation | Credit notes removed from working set | BR-03; exclusion reason recorded internally |
| 2.5 | For each remaining invoice: check PO-linkage field | Automation | Per-invoice PO-link flag set | Loop start — one invoice at a time |
| 2.6 | [DECISION] Is the PO-linkage field populated with a valid linked PO (not merely text in description)? | Automation | Invoice marked "PO linked" or "PO missing" | BR-01, BR-02; description-only text → "PO missing" |
| 2.7 | Retain "PO missing" invoices in qualifying set; discard "PO linked" invoices | Automation | Qualifying set updated | Loop end — repeat 2.5–2.7 for each invoice |
| 3.1 | Count qualifying invoices: invoice_count = size of qualifying set | Automation | invoice_count integer set | BR-05 |
| 3.2 | [DECISION] Is invoice_count = 0? | Automation | Branch: clean day or notify | BR-07 |
| 3.3 | Clean-day path (invoice_count = 0): [SME REVIEW] send congratulatory Slack message OR send nothing | Slack | Clean-day action taken | Source section 7 says send congrats; BR-007 says send nothing — SME must decide; see OQ-02 |
| 3.4 | Notify path (invoice_count > 0): build Coupa filtered URL using window_start and window_end | Automation | coupa_url string composed | URL pattern: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` [SME REVIEW — confirm base host for production] |
| 3.5 | Compose Slack Block Kit JSON payload using placeholders: invoice_count, coupa_url, window_start, window_end, run_date | Automation | Block Kit JSON payload ready | Title, three icon-led body lines, primary button, footer — see section 11 for template detail; BR-05, BR-06 |
| 4.1 | Send Slack Block Kit direct message to SME Slack member ID WLX9BD8FN | Slack | Message delivered to SME | BR-06; addressed by member ID, not email; HTTP Request activity with Block Kit JSON in body |
| 4.2 | Confirm message delivery (HTTP 200 / Slack API ok:true) | Slack | Delivery confirmed | If delivery fails → log failure; run ends |
| 5.1 | End process — log run as successful | Automation / Orchestrator | Run record complete | No Coupa records modified; BR-10 |

## 7. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|---|---|---|---|---|---|
| Coupa | API or web UI [SME REVIEW] | [SME REVIEW — REST API recommended; confirm endpoint base URL and auth type] | [SME REVIEW — OAuth2 / API key / basic] | Stored in UiPath Credential Store [DEFAULT] | Read-only access required; base host seen in source: uipath-test.coupahost.com — confirm production host |
| Slack | HTTP API (Slack Web API) | HTTPS POST to chat.postMessage | Bot token (OAuth2) | Bot token stored in UiPath Credential Store [DEFAULT] | Recipient addressed by member ID WLX9BD8FN; Block Kit payload in body; confirm bot has DM permission to WLX9BD8FN |
| UiPath Orchestrator | Internal (schedule trigger) | Orchestrator schedule | Robot credential | Managed by Orchestrator [DEFAULT] | Weekday trigger at 10:00 Romania time; timezone configured on trigger |

## 8. Business Rules

| ID | Rule | Source | Applies at step |
|---|---|---|---|
| BR-01 | Apply the no-PO-no-pay policy: invoices without a properly linked purchase order are flagged | BR-001 | 2.6 |
| BR-02 | A PO number present only in the invoice description field does not satisfy the PO-linkage requirement | BR-002 | 2.6 |
| BR-03 | Credit notes are excluded from the qualifying population before PO-linkage evaluation | BR-003 | 2.4 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven calendar days from run date | BR-004 | 1.2, 2.1 |
| BR-05 | Report the total count of qualifying invoices; individual invoice details are not listed in the message | BR-005 | 3.1, 3.5 |
| BR-06 | The Slack notification is sent to SME Irina Capatina (Slack member ID WLX9BD8FN) as a direct message and includes: the count, a no-PO-no-pay rationale, a remediation request, and a filtered Coupa list link | BR-006 | 4.1 |
| BR-07 | When a successful query returns zero qualifying invoices, no Slack notification is sent (clean-day path; see OQ-02 for conflict with section 7) | BR-007 | 3.2, 3.3 |
| BR-08 | No retry, fallback or error-recovery behaviour is required; a run that cannot complete is reported as a failed run | BR-008 | 2.3 |
| BR-09 | A failed run produces no Slack notification; no second notification path exists | BR-009 | 2.3 |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | 5.1 |

## 9. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|---|---|---|---|---|
| B1 | Credit note encountered | 2.4 | Invoice type field equals credit note | Exclude record from working set; log exclusion reason; continue processing |
| B2 | Description-only PO | 2.6 | PO number present in description field but PO-linkage field is empty or not a valid linked PO | Treat as missing PO; include invoice in qualifying set if other rules pass; BR-02 |
| B3 | Clean day — zero qualifying invoices | 3.2 | invoice_count = 0 after all filters applied | Per BR-07: send nothing; per source section 7: send congratulatory Slack message — see OQ-02 for resolution |
| B4 | Coupa response invalid | 2.2 | Coupa query returns error, non-parseable body, or timeout | Mark run as failed; no Slack notification sent; log failure; BR-08, BR-09 |

## 10. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|---|---|---|---|---|---|
| S1 | Coupa application unresponsive | HTTP timeout or connection refused on Coupa query | High | None (BR-08) [DEFAULT] | Log failure; mark run failed; end process |
| S2 | Coupa API/element not found | Unexpected response structure or missing invoice fields | High | None (BR-08) [DEFAULT] | Log failure; mark run failed; end process |
| S3 | Slack API failure | chat.postMessage returns non-200 or ok:false | High | None (BR-08) [DEFAULT] | Log failure; mark run failed; no second message attempt |
| S4 | Network timeout | General network connectivity loss during run | High | None (BR-08) [DEFAULT] | Log failure; mark run failed; end process |
| S5 | Credential expiry | Coupa or Slack credential rejected at authentication | High | None (BR-08) [DEFAULT] | Log failure; alert operations via Orchestrator alert [DEFAULT]; mark run failed |
| S6 | Unhandled exception | Any unhandled runtime exception | High | None (BR-08) [DEFAULT] | Log exception with stack trace; mark run failed; no notification sent |

## 11. Data Definitions

| Field | Type | Source | Target | Validation | Required |
|---|---|---|---|---|---|
| invoice_date | Date | Coupa invoice header | Filter input | Must fall within window_start to window_end inclusive; BR-04 | Yes |
| status | String (enum) | Coupa invoice header | Filter input | Must be "draft" or "new"; any other value excludes the record; BR-04 | Yes |
| invoice_type | String (enum) | Coupa invoice header | Filter input | If "credit note", record excluded; BR-03 | Yes |
| po_link | Reference / ID | Coupa invoice line(s) | Filter input | Must be a valid linked PO reference, not free text; empty or description-only → "PO missing"; BR-01, BR-02 | Yes |
| invoice_id | String / Integer | Coupa invoice header | Internal | Used to deduplicate and count qualifying records | Yes |
| window_start | Date | Automation (calculated) | Coupa query, Slack message | run_date minus 7 calendar days | Yes |
| window_end | Date | Automation (calculated) | Coupa query, Slack message | run_date (today) | Yes |
| run_date | Date | Automation (system clock) | Slack message footer | ISO date of execution | Yes |
| invoice_count | Integer | Automation (calculated) | Slack message | Count of records passing all filters; >= 0 | Yes |
| coupa_url | String (URL) | Automation (constructed) | Slack message button | Filtered Coupa URL using window_start, window_end, status=draft; confirm production base host [SME REVIEW] | Yes |
| slack_member_id | String | Hardcoded (source) | Slack API channel param | WLX9BD8FN — Irina Capatina; validate against Slack workspace before deploy | Yes |

**Slack Block Kit message placeholders**

| Placeholder | Resolved value |
|---|---|
| `{{invoice_count}}` | invoice_count integer |
| `{{coupa_url}}` | Constructed filtered Coupa URL |
| `{{window_start}}` | window_start date (display format [SME REVIEW]) |
| `{{window_end}}` | window_end date (display format [SME REVIEW]) |
| `{{run_date}}` | run_date date (display format [SME REVIEW]) |

**Block Kit message structure (from source section 4.3)**

| Element | Content |
|---|---|
| Title | `:receipt: {{invoice_count}} invoices need a purchase order` |
| Body line 1 | `:warning: {{invoice_count}} invoices from the last seven days have no purchase order linked.` |
| Body line 2 | `:no_entry: An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed.` |
| Body line 3 | `:point_right: Please make sure a purchase order exists for these invoices and is correctly linked to each one.` |
| Button | `Open the list in Coupa` — styled primary — links to `{{coupa_url}}` |
| Footer | `:calendar: Invoices dated {{window_start}} to {{window_end}} · :robot_face: No-PO Invoice Chaser · checked {{run_date}}` |
| Recipient | Slack direct message to member ID WLX9BD8FN |

## 12. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Retry policy conflict.** [SME REVIEW] Section 4.4 and BR-08 say no retry is needed, but image2 (future-state diagram) shows Coupa query retry and Slack message retry up to 3 attempts — confirm which applies before build.
2. **OQ-02 - Clean-day notification conflict.** [SME REVIEW] BR-007 says send nothing on a clean day; section 7 says send a congratulatory Slack message — SME must decide the authoritative rule.
3. **OQ-03 - Coupa access method.** [SME REVIEW] Confirm whether the automation uses the Coupa REST API or the web UI, and provide endpoint base URL, auth type, and credentials for the production environment.
4. **OQ-04 - Coupa production host.** [SME REVIEW] Source shows `uipath-test.coupahost.com` — confirm the production hostname for the Coupa URL in the Slack button.
5. **OQ-05 - PO-linkage field name.** [SME REVIEW] Confirm the exact Coupa API field or UI column name that holds the proper PO link on invoice lines (source says linkage is on lines, not the header).
6. **OQ-06 - Date display format.** [SME REVIEW] Confirm preferred display format for window_start, window_end and run_date in the Slack message (e.g., DD/MM/YYYY, YYYY-MM-DD).
7. **OQ-07 - Slack bot permissions.** [SME REVIEW] Confirm the Slack bot has `chat:write` scope and permission to open a DM with member ID WLX9BD8FN.
8. **OQ-08 - Orchestrator timezone config.** [DEFAULT] Orchestrator trigger timezone will be set to Europe/Bucharest to cover both EET (UTC+2) and EEST (UTC+3) automatically.
9. **Block Kit JSON verbatim.** Source states the exact Block Kit JSON is recorded in section 4 architectural considerations, but no JSON is present in the source document — treat as a missing input; the SDD must obtain or reconstruct it.
10. **Appendix A diagrams.** Image1 (current-state map) and image2 (future-state map) were readable from PNG files; no unreadable EMF/WMF files were present.
11. **No appendix attachment content.** Source references section 4 architectural considerations for the verbatim Block Kit JSON, but that section's content was not included in the provided document.

## 13. Success Criteria

1. A weekday test run executes at exactly 10:00 Romania time and completes without error.
2. Only invoices with status draft or new and invoice date within the past seven days are included in evaluation.
3. All credit-note records are excluded from the qualifying set.
4. An invoice with a PO number in the description field but no valid linked PO is counted as a missing-PO invoice.
5. The Slack message reports the correct total count and contains a working filtered Coupa link covering the same date window.
6. Exactly one Slack direct message is delivered to Slack member ID WLX9BD8FN when qualifying invoices exist.
7. No Slack message is sent when the qualifying count is zero (pending OQ-02 resolution).
8. No Coupa record is created, modified, or deleted during any test or production run.
