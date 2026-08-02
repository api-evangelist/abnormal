---
name: Investigate an Abnormal account takeover case
description: Work an Abnormal case end to end — list cases, read the AI-generated analysis and event timeline, pivot to the employee's identity and login history, act on the case, and verify the action.
api: openapi/abnormal-client-api-openapi-original.yml
operations:
  - v1_cases_retrieve
  - v1_cases_retrieve_2
  - v1_cases_analysis_retrieve
  - v1_cases_create
  - v1_cases_actions_retrieve
  - v1_employee_retrieve
  - v1_employee_identity_retrieve
  - v1_employee_logins_retrieve
generated: '2026-08-02'
method: generated
source: openapi/abnormal-client-api-openapi-original.yml
---

# Investigate an Abnormal account takeover case

An Abnormal **case** correlates the signals that indicate a compromised account. The case carries Abnormal's own analysis; the employee endpoints are how you pivot from the case to the human.

## Before you start

Same auth and region rules as every Abnormal operation: `Authorization: Bearer <ACCESS_TOKEN>`, source IP on the Portal allowlist, US or EU host per tenant. Send `Mock-Data: True` while developing.

## Steps

1. **List cases.** `v1_cases_retrieve` (`GET /cases`) with `filter`, `pageSize` and `pageNumber`. Filter on `lastModifiedTime` to pick up cases that changed since your last poll.
2. **Page to exhaustion** on `nextPageNumber`.
3. **Read the case.** `v1_cases_retrieve_2` (`GET /cases/{case_id}`).
4. **Read Abnormal's analysis.** `v1_cases_analysis_retrieve` (`GET /cases/{case_id}/analysis`) returns `CaseAnalysis` — an `insights` array (each a `signal` plus a `description`, which is the explanation to surface to an analyst) and an `eventTimeline`. This is the field to quote in a ticket, not a re-derived summary.
5. **Pivot to the employee.** `v1_employee_retrieve` (`GET /employee/{email_address}`) returns name, title and manager. `v1_employee_identity_retrieve` (`GET /employee/{email_address}/identity`) returns the behavioural genome attributes. `v1_employee_logins_retrieve` (`GET /employee/{email_address}/logins`) returns the login history that corroborates or refutes the takeover.
6. **Act on the case.** `v1_cases_create` (`POST /cases/{case_id}`) submits the case action.
7. **Verify.** Poll `v1_cases_actions_retrieve` (`GET /cases/{case_id}/actions/{action_id}`) until terminal.

## Rules

- Step 6 has no idempotency key. Always run step 7 before resubmitting after a timeout.
- The email address in step 5 is a path parameter — URL-encode the `@`.
- A 404 on an employee endpoint means the address is not known to this tenant, not that the API is wrong; do not retry it.
- Keep the case-action and threat-action loops separate. Remediating the messages in a threat (see `abnormal-triage-and-remediate-threats.md`) does not close the case, and closing the case does not remediate messages.
