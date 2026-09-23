# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-773 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-773 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Replace a daily manual Coupa review with a fully automated weekday run that identifies invoices lacking a properly linked purchase order and notifies the AP responsible on Slack, enforcing the no-PO-no-pay policy consistently.

**Owning department:** Accounts Payable

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack member ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Property | Value |
|----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable – invoice compliance |
| Short description | Queries Coupa daily for invoices (status draft/new, past 7 days) with no linked PO, then sends a Slack summary count to the AP SME; sends nothing on a clean day |
| Required roles | Unattended robot; SME receives Slack notification |
| Trigger and schedule | Weekday schedule, 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | ~1 run per weekday; live sample shows up to 194 qualifying invoices per run |
| Average handling time (manual vs automated target) | Manual: ~daily ad-hoc effort; Automated: seconds per run [SME REVIEW exact manual AHT] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low – credit notes and description-only POs are deterministic exclusions; no qualifying invoices is a clean outcome |
| Input data | Coupa invoice list (status, invoice date, PO linkage on invoice lines, invoice type) |
| Output data | One Slack Block Kit DM to SME with qualifying invoice count and filtered Coupa URL; or no message on a clean day |

## 4. To-Be Process (High Level)

The automation runs as an unattended process every weekday at 10:00 Romania time. It reads the Coupa invoice list, applies the approved status, date-window, PO-linkage and credit-note rules, counts qualifying invoices, and delivers one Slack direct message to the SME. It does nothing else.

**Manual steps eliminated:**
- Manual Coupa login and filter configuration
- Record-by-record review for PO linkage
- Manual field copying and message preparation
- Manual Slack message composition and sending

**What stays human:** The SME reviews the notification and arranges PO creation and linkage for flagged invoices. Requester follow-up tracking remains out of scope.

**Automation boundary:** Read, filter, count, notify. No PO creation, no invoice approval, no Coupa record modification, no supplier communication.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Trigger: weekday schedule fires at 10:00 Romania time | Scheduler | Run context initialised | Weekday only (Mon–Fri); timezone = EET/EEST. BR-04 |
| 1.2 | Calculate date window: window_end = today's date; window_start = today − 7 days | Automation | window_start and window_end variables set | Used in Coupa query and in Slack message footer |
| 2.1 | Authenticate to Coupa | Coupa | Authenticated session or token obtained | Credential stored in credential store [SME REVIEW auth method: UI login vs API key/OAuth] |
| 2.2 | Query Coupa invoice list filtered to status = draft OR new AND invoice_date >= window_start AND invoice_date <= window_end | Coupa | Raw list of invoices in the date/status window | BR-04; PO linkage is on invoice lines, not the header – must retrieve line-level data |
| 2.3 | Exclude records where invoice type = credit note | Automation | Credit notes removed from working set | BR-03; log count of excluded credit notes |
| 2.4 | For each remaining invoice: inspect PO linkage on invoice lines | Automation (loop) | Per-invoice flag: linked or not linked | Loop boundary: START – iterate over filtered invoice list |
| 2.5 | Determine PO linkage status: linked = at least one line has a properly linked PO object; not linked = no linked PO OR PO number present only in description field | Automation | Invoice classified as qualifying (no linked PO) or non-qualifying | BR-01, BR-02; description-only PO text does NOT satisfy linkage requirement |
| 2.6 | Add qualifying invoice to count; discard non-qualifying | Automation | Running count of invoices with no linked PO | Loop boundary: END |
| 3.1 | Evaluate count: if count = 0, go to step 3.2; if count > 0, go to step 4.1 | Automation | Branch decision | BR-07 |
| 3.2 | Count = 0: send clean-day Slack message congratulating that all invoices have a PO assigned | Slack | Slack DM delivered to SME (WLX9BD8FN) | Source section 7 specifies a congratulatory message for the zero-match case; contradicts BR-07 which says send nothing – [SME REVIEW: confirm desired zero-match behaviour] |
| 4.1 | Build Coupa filtered URL: base URL + invoice_date_gteq=window_start + invoice_date_lteq=window_end + status_eq=draft | Automation | coupa_url string constructed | Example base: https://uipath-test.coupahost.com/invoices with query params. [SME REVIEW: confirm production Coupa hostname] |
| 4.2 | Compose Slack Block Kit message payload: populate {{invoice_count}}, {{coupa_url}}, {{window_start}}, {{window_end}}, {{run_date}} placeholders | Automation | Complete Block Kit JSON payload ready | Title, three body lines with emoji, primary button "Open the list in Coupa", footer with dates and run date |
| 4.3 | Send Slack DM to SME Slack member ID WLX9BD8FN via Slack HTTP Request activity | Slack | Slack API returns HTTP 200; DM delivered | Recipient identified by member ID, not email. BR-06 |
| 4.4 | Log run outcome: count of qualifying invoices, window used, run timestamp, Slack delivery status | Automation | Run log entry written | Enables distinction between clean result and technical failure |
| 5.1 | End process | Automation | Process terminates cleanly | No retry, no fallback per BR-08 |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|---------  |
| Coupa | Web / API | [SME REVIEW: UI automation or REST API] | [SME REVIEW: UI credential or API key/OAuth] | Credential store [SME REVIEW] | PO linkage is on invoice lines; query must retrieve line-level data. Production hostname [SME REVIEW]; test host uipath-test.coupahost.com seen in source |
| Slack | API (HTTP) | Slack HTTP Request activity | Bot token / OAuth | Credential store [SME REVIEW] | DM to member ID WLX9BD8FN; Block Kit payload; protocol = Slack Web API over HTTPS |
| Weekday Scheduler | Schedule trigger | UiPath Orchestrator | n/a | n/a | 10:00 EET/EEST, Mon–Fri |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|-----------------|
| BR-01 | Invoices without a properly linked purchase order are subject to the no-PO-no-pay policy and must be flagged. | BR-001 | 2.5 |
| BR-02 | A PO number present only in the invoice description field does not satisfy the PO-linkage requirement. | BR-002 | 2.5 |
| BR-03 | Credit notes must be excluded from the qualifying population before counting. | BR-003 | 2.3 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days. | BR-004 | 2.2 |
| BR-05 | The message reports the total count of qualifying invoices; individual invoices are not listed. | BR-005 | 4.2 |
| BR-06 | Deliver the count to SME Irina Capatina (Slack member ID WLX9BD8FN) by Slack DM, with the no-PO-no-pay rationale, a remediation request, and a filtered Coupa link. | BR-006 | 4.3 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices. | BR-007 | 3.1 |
| BR-08 | No retry, fallback or error-recovery behaviour is required; a run that cannot complete is recorded as a failed run. | BR-008 | 5.1 |
| BR-09 | A run that cannot complete produces no notification. | BR-009 | 5.1 |
| BR-10 | The automation must not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion. | BR-010 | All steps |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|------------------|--------|
| B1 | Credit note encountered | 2.3 | Invoice type = credit note | Exclude record from working set; log exclusion count; continue loop |
| B2 | Description-only PO | 2.5 | PO number present only in description field, no line-level PO object linked | Treat as no linked PO; include in qualifying count if other rules pass (BR-02) |
| B3 | No qualifying invoices (clean day) | 3.1 | Count = 0 after filtering | Send congratulatory Slack DM to SME per source section 7 [SME REVIEW: confirm vs BR-07 which says send nothing] |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-----------------|----------|-------------|--------|
| S1 | Coupa unavailable | Coupa login or query fails at step 2.1–2.2 | High | None (BR-08) [DEFAULT] | Log failure; mark run as failed; no notification sent (BR-09) |
| S2 | Coupa element not found | UI element or API endpoint not found during invoice query | High | None [DEFAULT] | Log failure with element detail; mark run as failed |
| S3 | Slack delivery failure | Slack HTTP Request returns non-200 at step 4.3 | High | None [DEFAULT] | Log HTTP status and error body; mark run as failed |
| S4 | Network timeout | HTTP request times out reaching Coupa or Slack | Medium | None [DEFAULT] | Log timeout; mark run as failed |
| S5 | Credential expiry | Credential store returns empty or invalid credential | High | None [DEFAULT] | Log credential error; alert [SME REVIEW: alert target] |
| S6 | Unhandled exception | Unexpected runtime error anywhere in process | High | None [DEFAULT] | Log full exception; mark run as failed; no notification sent |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Zero-match behaviour.** Source section 7 says send a congratulatory Slack message on a clean day; BR-007 says send nothing – [SME REVIEW] confirm which applies.
2. **OQ-02 - Coupa access method.** [SME REVIEW] Confirm whether the robot accesses Coupa via UI automation or the Coupa REST API; the choice affects authentication and PO-line retrieval design.
3. **OQ-03 - Coupa production hostname.** [SME REVIEW] Confirm the production Coupa hostname; only the test host (uipath-test.coupahost.com) is named in the source.
4. **OQ-04 - Coupa URL status param.** Source example URL uses status_eq=draft only; BR-04 includes status new as well – [SME REVIEW] confirm whether the Coupa URL filter must cover both statuses.
5. **OQ-05 - Slack bot credential.** [SME REVIEW] Confirm the Slack bot token / OAuth app name and the credential store location.
6. **OQ-06 - Exact manual AHT.** [SME REVIEW] Provide the average manual handling time per run for the process overview baseline.
7. **OQ-07 - FTE effort.** [SME REVIEW] Provide the FTE fraction currently spent on this daily review.
8. **OQ-08 - Block Kit JSON payload.** Source states the exact Block Kit JSON is recorded in architectural considerations section 4 of the source document – that section is not present in the converted file; [SME REVIEW] supply or confirm the payload.
9. **Weekday schedule timezone.** Romania time is EET (UTC+2) / EEST (UTC+3); Orchestrator trigger must be configured with the correct timezone ID. [DEFAULT]
10. **No retry by design.** BR-08 explicitly prohibits retry and fallback; all system errors result in a failed run with no notification. [DEFAULT]
11. **PO linkage on lines.** Coupa holds PO linkage at the invoice line level, not the header; the query and linkage check must retrieve line-level data. [DEFAULT inference from source]

## 11. Success Criteria

1. A weekday test run triggers at exactly 10:00 Romania time and completes without manual intervention.
2. Only invoices with status draft or new and invoice_date within the past seven calendar days are evaluated.
3. Credit notes are excluded from the count; a description-only PO text does not satisfy the linkage check.
4. The Slack DM is delivered to member ID WLX9BD8FN with the correct invoice count, the no-PO-no-pay rationale, and a working filtered Coupa link.
5. When the qualifying count is zero, behaviour matches the SME-confirmed rule from OQ-01 (either no message or congratulatory message).
6. A run that cannot complete is recorded as a failed run; no Slack notification is sent and no Coupa record is modified.
