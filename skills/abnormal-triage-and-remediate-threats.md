---
name: Triage and remediate Abnormal threats
description: Page the Abnormal Threat Log for a time window, pull full threat detail with its messages, links and attachments, take a remediation action, and confirm the action landed.
api: openapi/abnormal-client-api-openapi-original.yml
operations:
  - v1_threats_retrieve
  - v1_threats_retrieve_2
  - v1_threats_create
  - v1_threats_actions_retrieve
  - v1_threats_links_retrieve
  - v1_threats_attachments_retrieve
  - v1_messages_remediation_history_retrieve
  - v1_threats_export_csv_retrieve
generated: '2026-08-02'
method: generated
source: openapi/abnormal-client-api-openapi-original.yml
---

# Triage and remediate Abnormal threats

The Threat Log is Abnormal's top-level unit of detection: one **threat** is an attack campaign, and it contains one or more **messages** (individual emails, each with an `abxMessageId`).

## Before you start

- Authenticate with `Authorization: Bearer <ACCESS_TOKEN>`. The token is issued per organization in the Abnormal Portal under **Settings > Integrations > Abnormal REST API**. There are no scopes.
- Your egress IP must be in the organization's Portal allowlist, or every call returns **403** regardless of the token.
- Use the US host `https://api.abnormalplatform.com/v1` unless the tenant is EU, in which case use `https://eu.rest.abnormalsecurity.com/v1`.
- While building, send `Mock-Data: True` to get the documented example payloads back instead of live tenant data.

## Steps

1. **List threats for a window.** `v1_threats_retrieve` (`GET /threats`). Supply a `filter` expression — a `receivedTime` range using `gte`/`lte` — plus `pageSize` and `pageNumber`. A range wider than the endpoint allows returns **400** (`InvalidDateError`); a malformed filter returns **404**.
2. **Page to exhaustion.** Read `nextPageNumber` from the response and re-request until it is absent or null. Note that `pageNumber`/`nextPageNumber` are only returned when a `filter` was supplied — without one the response is unpaged.
3. **Pull threat detail.** `v1_threats_retrieve_2` (`GET /threats/{threat_id}`) returns `ThreatDetails` with the `messages` array. Each `ThreatMessage` carries `abxMessageId`, `subject`, `fromAddress`, `recipientAddress`, `receivedTime`, `remediationStatus` and `abxPortalUrl` (the deep link back into the Portal).
4. **Enrich before acting.** `v1_threats_links_retrieve` (`GET /threats/{threat_id}/links`) returns the URLs found in the campaign with `linkType` and `domainLink`; `v1_threats_attachments_retrieve` (`GET /threats/{threat_id}/attachments`) returns the attachment names. Both key back to `abxMessageId`.
5. **Take the action.** `v1_threats_create` (`POST /threats/{threat_id}`) submits a remediation action against the threat. It returns an action identifier rather than a completed result.
6. **Confirm it landed.** Poll `v1_threats_actions_retrieve` (`GET /threats/{threat_id}/actions/{action_id}`) until the action reports a terminal state. Cross-check the per-message outcome with `v1_messages_remediation_history_retrieve` (`GET /messages/{message_id}/remediation_history`).
7. **Bulk export, if you need it.** `v1_threats_export_csv_retrieve` (`GET /threats_export/csv`) returns `text/csv` rather than JSON.

## Rules

- **There is no idempotency key.** Step 5 is not safe to blindly retry. If a `POST /threats/{threat_id}` times out, run step 6 first and only resubmit if no action exists.
- **429 means concurrency, not a quota.** 61 of the 67 operations declare 429, and Abnormal describes the limit as concurrent requests per resource type per token. Serialize the page loop in step 2 rather than fanning it out, and back off exponentially. No `Retry-After` header is published.
- **403 is usually the allowlist, not the token.** Distinguish it from 401 (missing or invalid token) before rotating credentials.
- Errors come back as `application/json` `{"error": "<message>"}` — not RFC 9457 problem+json. See `errors/abnormal-problem-types.yml`.
