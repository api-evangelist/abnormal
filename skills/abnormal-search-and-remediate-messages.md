---
name: Search and remediate messages across the mail estate
description: Run an asynchronous message search across the tenant, poll it to completion, download the evidence, and submit a bulk remediation over the matched messages.
api: openapi/abnormal-client-api-openapi-original.yml
operations:
  - v1_search_create
  - v1_search_activities_retrieve
  - v1_search_activities_status_retrieve
  - v1_search_messages_eml_retrieve
  - v1_search_messages_attachments_download_retrieve
  - v1_search_remediate_create
generated: '2026-08-02'
method: generated
source: openapi/abnormal-client-api-openapi-original.yml
---

# Search and remediate messages across the mail estate

This is the only genuinely asynchronous surface in the Abnormal API: search and remediate both return **202 Accepted** and hand back an activity you must poll.

## Steps

1. **Submit the search.** `v1_search_create` (`POST /search`). Returns **202 Accepted** with an activity identifier — not results.
2. **Poll for completion.** `v1_search_activities_status_retrieve` (`GET /search/activities/{activity_log_id}/status`) until it reports a terminal state. Do not poll tighter than a few seconds; 429 here is concurrency-based and will stall the whole loop.
3. **List your activities.** `v1_search_activities_retrieve` (`GET /search/activities`) enumerates search and remediation activities — use it to recover an activity id you lost, and to detect a duplicate submission before you make one.
4. **Pull evidence.** `v1_search_messages_eml_retrieve` (`GET /search/messages/{message_id}/eml`) returns the raw email as `message/rfc822`. `v1_search_messages_attachments_download_retrieve` (`GET /search/messages/attachments/download`) returns attachments as `application/octet-stream`. Neither is JSON.
5. **Remediate the matched set.** `v1_search_remediate_create` (`POST /search/remediate`). Also returns **202 Accepted**.
6. **Poll the remediation** with `v1_search_activities_status_retrieve`, same as step 2.

## Rules

- **This is the highest-risk flow in the API and it has no idempotency key.** A retried `POST /search/remediate` is a second bulk remediation, not a no-op. Before any resubmit, call step 3 and confirm no equivalent activity is already running or complete.
- 202 is success. Do not treat a missing result body as a failure.
- Steps 4's responses are binary; stream them to disk rather than buffering and decoding.
- Long-running exports and downloads are the operations that declare **502**; treat a 502 here as transient and retry the *read*, never the submit.
