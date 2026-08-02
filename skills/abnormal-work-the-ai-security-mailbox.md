---
name: Work the Abnormal AI Security Mailbox
description: Process user-reported phishing — list reported campaigns and their judgements, catch the reports Abnormal could not analyze, and file a Detection 360 report when Abnormal got a verdict wrong.
api: openapi/abnormal-client-api-openapi-original.yml
operations:
  - v1_abusecampaigns_retrieve
  - v1_abusecampaigns_retrieve_2
  - v1_abuse_mailbox_not_analyzed_retrieve
  - v1_detection360_reports_create
  - v1_detection360_reports_retrieve
  - v1_messages_download_retrieve
generated: '2026-08-02'
method: generated
source: openapi/abnormal-client-api-openapi-original.yml
---

# Work the Abnormal AI Security Mailbox

The AI Security Mailbox (formerly Abuse Mailbox) is the employee-reporting channel. Reports are grouped into **campaigns**; each campaign carries Abnormal's `judgementStatus` and `overallStatus`.

## Steps

1. **List reported campaigns.** `v1_abusecampaigns_retrieve` (`GET /abusecampaigns`) with `filter` on the reporting time range, plus `pageSize`/`pageNumber`. Follow `nextPageNumber` until null.
2. **Read a campaign.** `v1_abusecampaigns_retrieve_2` (`GET /abusecampaigns/{campaign_id}`) returns `AbuseCampaignDetails`: `firstReported`, `lastReported`, `messageId`, `subject`, `fromName`, `fromAddress`, `recipientName`, `recipientAddress`, `judgementStatus`, `overallStatus`, `attackType`.
3. **Catch the gaps.** `v1_abuse_mailbox_not_analyzed_retrieve` (`GET /abuse_mailbox/not_analyzed`) returns reports Abnormal did **not** analyze, each with a `not_analyzed_reason`, `reporter`, `recipient` and `reported_datetime`. These never appear in step 1, so an integration that only reads campaigns silently drops them — always poll both.
4. **Pull the raw email when you need it.** `v1_messages_download_retrieve` (`GET /messages/{message_id}/download`) returns the message body. Note the media type is not JSON.
5. **Dispute a verdict.** `v1_detection360_reports_create` (`POST /detection360/reports`) files a Detection 360 report — Abnormal's false-positive / false-negative channel. Supply the `inquiry_type` and the messages in question.
6. **Track the dispute.** `v1_detection360_reports_retrieve` (`GET /detection360/reports`) lists submitted reports with `status`, `submission_datetime`, `submitted_by` and, once analyzed, a `report` containing `analysis` and `root_causes`.

## Rules

- Step 5 has no idempotency key and no natural dedupe key. Read step 6 first and match on `submission_datetime` + message id before submitting again, or you will file duplicate Detection 360 reports.
- `judgementStatus` and `overallStatus` are distinct — one is the verdict on the report, one is the campaign's overall state. Do not collapse them into a single field in downstream systems.
- Steps 4 returns a binary body; do not attempt to JSON-decode it.
