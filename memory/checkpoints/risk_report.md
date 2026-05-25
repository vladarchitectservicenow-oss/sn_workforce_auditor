# SN Workforce Auditor — Risk Report

## Risk Register

| ID | Category | Risk | Likelihood | Impact | Severity | Mitigation |
|----|----------|------|-----------|--------|----------|------------|
| R01 | Data Integrity | ServiceNow `task` table returns incomplete fields (missing `resolved_by`, null `reassignment_count`) | High | High | P0 | Code defensively handles null/missing fields: `resolved_by` → None, `reassignment_count=""` → 0. All optional fields parsed with `or None` fallback. |
| R02 | API Availability | ServiceNow REST API throttles or returns 429/503 during bulk audit | Medium | High | P1 | Single API call per audit (not paginated). Timeout set to 30 seconds. On failure, exception is caught and empty list returned — audit won't crash but will report zero tasks. |
| R03 | State Mapping | New ServiceNow releases introduce state codes not in normalization table | Low | Medium | P2 | Unknown states pass through unchanged (no classification). Discrepancies are only flagged for known resolved/cancelled states. New state codes won't produce false positives. |
| R04 | Authentication | Basic Auth credentials exposed in CLI arguments visible in `ps aux` | Medium | High | P1 | Credentials are CLI args only (not environment variables, not config files). Risk accepted for audit tooling — auditors run in controlled environments. Mitigation: run with `ps` protections or use `--password-file` in future release. |
| R05 | Rate Limiting | Audit of 1000+ tasks in one window hits ServiceNow API pagination limit | Medium | Medium | P2 | `sysparm_limit=1000` caps the query. For instances with >1000 tasks in window, results will be truncated silently. Future release should implement pagination cursor. |
| R06 | Timezone Drift | `datetime.now(timezone.utc)` differs from ServiceNow server timezone | Medium | Medium | P2 | Window computed in UTC. ServiceNow stores timestamps in GMT. If instance uses non-GMT timezone, boundary tasks may be missed. Mitigation: document assumption of GMT-aligned instance. |
| R07 | Schema Drift | ServiceNow REST API field names change across releases (e.g., `reassignment_count` renamed) | Low | High | P1 | Field names are hardcoded in fetch_tasks(). If a field is renamed, `parse_task()` returns empty/default values for that field — audit still runs but accuracy degrades. Mitigation: validate field presence via API metadata call before audit. |
| R08 | False Positives | Bot correctly resolves task, but `resolved_by` field is null due to platform bug | Medium | Medium | P2 | The `unattributed_resolution` discrepancy is severity "Moderate", not "Critical". Human review step in SOP mitigates false alarms. |
| R09 | Instance Hibernation | ServiceNow PDI hibernates during automated audit, causing connection errors | Medium | High | P1 | HTTP timeout set to 30s prevents hang. Connection failure → empty results → report shows zero tasks with error logged. Cron job should wake PDI before audit. |
| R10 | Output Overwrite | `--output audit_report.json` overwrites previous audit without warning | High | Low | P3 | CLI writes to specified output file, overwriting existing. If auditors run sequential audits without changing output path, results are lost. Mitigation: use `--output audit_report_$(date +%Y%m%d).json` convention. |
| R11 | Credential Leak | Audit report JSON accidentally contains credentials if API response includes them | Low | High | P1 | Report only includes structured fields (sys_id, number, state, etc.) — never raw API response. Credential leak from report is structurally impossible. |
| R12 | Python Version | Host system runs Python < 3.10 (e.g., older RHEL 7 with Python 3.6) | Low | Medium | P3 | `dataclasses` requires Python 3.7+. Tool will fail with `ModuleNotFoundError`. Mitigation: document Python 3.10+ requirement in README and SOP. |

## Risk Summary

| Severity | Count | Description |
|----------|-------|-------------|
| P0 (Critical) | 1 | R01 — Data integrity: null fields from API |
| P1 (High) | 5 | R02, R04, R07, R09, R11 — API availability, credential exposure, schema drift, hibernation, credential leak |
| P2 (Medium) | 4 | R03, R05, R06, R08 — State mapping gaps, pagination, timezone drift, false positives |
| P3 (Low) | 2 | R10, R12 — Output overwrite, Python version mismatch |

## Risk Heatmap

```
Impact
  H │   · R07        · R01 R02
    │                 · R04 R09
    │                 · R11
  M │   · R03        · R05 R06
    │                 · R08
  L │   · R12        · R10
    └────────────────────────────
       L     M     H     Likelihood
```

## Acceptable Residual Risk

After mitigations applied:
- **R01** (P0) → mitigated by defensive parsing → residual risk: **Low**
- **R04** (P1) → accepted risk for audit tooling → residual risk: **Medium** (requires controlled execution environment)
- **R09** (P1) → timeout prevents hang → residual risk: **Low-Medium** (empty report returned instead of crash)
- All other risks mitigated to Low or are procedural (documented in SOP)
