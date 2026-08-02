---
name: Audit Abnormal security posture and portal access
description: Read the SPM v2 posture catalog, query postures by risk and benchmark, walk a posture's timeline, and reconcile it against portal RBAC roles, users and audit logs.
api: openapi/abnormal-client-api-openapi-original.yml
operations:
  - v1_spm_v2_posture_catalog_retrieve
  - v1_spm_v2_postures_query_create
  - v1_spm_v2_postures_retrieve
  - v1_spm_v2_postures_timeline_retrieve
  - v1_spm_v2_reports_summary_retrieve
  - v1_spm_v2_workflow_logs_raw_json_retrieve
  - v1_roles_retrieve
  - v1_users_retrieve
  - v1_auditlogs_retrieve
  - v1_security_settings_retrieve
generated: '2026-08-02'
method: generated
source: openapi/abnormal-client-api-openapi-original.yml
---

# Audit Abnormal security posture and portal access

Security Posture Management (SPM v2) is the newest surface in the API — it was added to the published spec in the 2026-05-28 revision of version 1.4.3. It uses `limit`/`offset` paging rather than the `pageSize`/`pageNumber` convention the rest of the API uses.

## Steps

1. **Get the catalog.** `v1_spm_v2_posture_catalog_retrieve` (`GET /spm-v2/posture-catalog`) returns every posture check Abnormal evaluates, with `posture_area`, `platform_type`, `benchmarks`, `risk_level`, `insight` and `remediation_steps`.
2. **Query current state.** `v1_spm_v2_postures_query_create` (`POST /spm-v2/postures/query`) takes a `PostureListParams` body — `risk_levels`, `statuses`, `benchmarks`, `last_evaluated_at`, `posture_area`, `platform_type`, `posture_types` — and returns `PostureListResponse`.
3. **Read one posture.** `v1_spm_v2_postures_retrieve` (`GET /spm-v2/postures/{posture_id}`) returns `PostureDetail` with `status`, `workflow_status`, `actor` and `risk_level`.
4. **Walk its history.** `v1_spm_v2_postures_timeline_retrieve` (`GET /spm-v2/postures/{posture_id}/timeline`).
5. **Roll it up.** `v1_spm_v2_reports_summary_retrieve` (`GET /spm-v2/reports/summary`) returns `PostureStats` — total, passing and failing counts. This is the board-level number.
6. **Get the raw evidence.** `v1_spm_v2_workflow_logs_raw_json_retrieve` (`GET /spm-v2/workflow-logs/{workflow_log_id}/raw-json`) when you need the underlying workflow log rather than the summarized posture.
7. **Reconcile against access.** `v1_roles_retrieve` (`GET /roles`) returns RBAC roles and their policies; `v1_users_retrieve` (`GET /users`) returns portal users with `role`, `resource_permissions`, `sso_enabled` and `local_login_enabled`; `v1_security_settings_retrieve` (`GET /security-settings`) returns session timeout configuration.
8. **Prove who changed what.** `v1_auditlogs_retrieve` (`GET /auditlogs`) returns portal audit events with `action`, `actionDetails`, `category`, `sourceIp`, `status`, `user` and `timestamp`. Supply a `filter` or the `pageNumber`/`nextPageNumber` fields will not be returned.

## Rules

- **Paging differs on this surface.** SPM v2 endpoints use `limit`/`offset`; the rest of the API uses `pageSize`/`pageNumber`. Do not share a paging helper across both.
- Step 2 is a `POST` that reads — it is a query, not a mutation. It is safe to retry, unlike every other `POST` in this API.
- `local_login_enabled: true` alongside `sso_enabled: true` on a portal user is the finding worth escalating; the API reports both, it does not judge them.
- Audit log retention and the maximum queryable window are not published in the spec; a range wider than the endpoint allows returns **400** (`InvalidDateError`), so narrow and retry rather than treating it as an error.
