# SN Workforce Auditor — Edge Cases

## Purpose

This document catalogs boundary conditions, corner cases, and unexpected inputs that the Workforce Auditor may encounter. Each edge case describes the scenario, expected behavior, and whether it's currently handled.

## Edge Case Catalog

### EC-01: Zero Tasks in Audit Window
**Scenario**: ServiceNow returns empty result set for the specified time window.  
**Expected Behavior**: Report generated with `total_tasks=0`, all rates at `0.0`, no discrepancies. No crash.  
**Handled**: ✅ Yes — `audit()` guards against division by zero.  
**Test Coverage**: TS-10

### EC-02: Single Task — All Fields Valid
**Scenario**: API returns exactly one task with complete data.  
**Expected Behavior**: Report with `total_tasks=1`. Rates computed correctly (either 0% or 100%).  
**Handled**: ✅ Yes — standard code path; no special casing needed.  
**Test Coverage**: Implicit in TS-11 (2-task case validates ratio math)

### EC-03: 1000 Tasks (Limit Boundary)
**Scenario**: API returns exactly 1000 tasks (the `sysparm_limit`).  
**Expected Behavior**: All 1000 tasks parsed and classified. Report includes all discrepancies.  
**Handled**: ✅ Yes — no internal hard limit; Python handles 1000 dataclass objects easily (<50 MB memory).  
**Risk**: Tasks beyond 1000 are silently truncated. Documented as R05 in risk_report.md.

### EC-04: Task with All Fields Null/Empty
**Scenario**: API returns a task record with every optional field as null or empty string.  
**Input**: `{"sys_id": "x1", "number": "INC001", "state": "1", "resolved_by": null, "opened_at": "", "closed_at": null, "reassignment_count": ""}`  
**Expected Behavior**: `parse_task()` returns Task with `resolved_by=None`, `closed_at=None`, `reassignment_count=0`, `opened_at=""`. Classification runs but produces zero discrepancies (state "1" is not in mapping).  
**Handled**: ✅ Yes — field-level `or None` / `or 0` fallbacks.

### EC-05: Task with Malformed State Code
**Scenario**: API returns a state code that is not a standard ServiceNow state (e.g., "-1", "999", "pending_approval").  
**Expected Behavior**: `normalize_state()` returns the code unchanged (pass-through). No discrepancies triggered because classification only fires for known "resolved"/"cancelled" states.  
**Handled**: ✅ Yes — mapping is a lookup; unknown keys pass through unchanged.  
**Test Coverage**: TS-06

### EC-06: Network Timeout
**Scenario**: ServiceNow instance is unreachable or takes >30 seconds to respond.  
**Expected Behavior**: `urllib.request.urlopen(timeout=30)` raises `URLError` or `socket.timeout`. Exception caught in `fetch_tasks()` → returns empty list. Report shows zero tasks.  
**Handled**: ✅ Yes — blanket `except Exception` in `fetch_tasks()` with return `[]`.  
**Risk**: Auditor can't distinguish "no tasks in window" from "network failure" — both produce zero-task report. Future: add error logging.

### EC-07: API Returns Non-JSON Response
**Scenario**: ServiceNow returns HTML error page or XML instead of JSON (e.g., instance hibernation page).  
**Expected Behavior**: `json.loads()` raises `JSONDecodeError`. Caught by generic `except Exception` → empty list returned.  
**Handled**: ✅ Yes — blanket exception handler.  
**Risk**: Same as EC-06 — indistinguishable from empty results.

### EC-08: resolved_by Contains Bot System User ID
**Scenario**: `resolved_by` is a sys_user sys_id string (e.g., "6816f79cc0a8016401c5a33be04be441") instead of a human-readable name.  
**Expected Behavior**: `resolved_by` is stored as-is. Classification logic only checks `truthiness` of `resolved_by` — a sys_id string is truthy, so it counts as "attributed resolution".  
**Handled**: ✅ Yes — resolved_by is treated as opaque string.  
**Note**: Future enhancement could resolve sys_id to display name via additional API call.

### EC-09: Window Spans DST Transition
**Scenario**: Audit window crosses a daylight saving time transition.  
**Expected Behavior**: UTC timestamps are unambiguous. ServiceNow stores timestamps in GMT. No DST issues.  
**Handled**: ✅ Yes — all times in UTC.

### EC-10: Duplicate Tasks (Same sys_id)
**Scenario**: API returns the same task twice (unlikely but possible with pagination bugs).  
**Expected Behavior**: Both copies are independently parsed and classified. Duplicate discrepancies appear in report.  
**Handled**: ⚠️ Partially — no deduplication logic. If duplicates occur, report inflates discrepancies. Mitigation: `sysparm_limit=1000` on a single query should not produce duplicates.  
**Risk**: Low — single-page query with no pagination makes duplicates unlikely.

### EC-11: Extremely Long Task Number
**Scenario**: Task number exceeds typical length (e.g., 500-character custom number).  
**Expected Behavior**: Stored as-is in `Task.number`. Report JSON may be larger but not broken.  
**Handled**: ✅ Yes — Python strings handle arbitrarily long values.

### EC-12: CLI Password with Special Characters
**Scenario**: Password contains shell-special characters like `$`, `!`, `#`, `%`.  
**Expected Behavior**: Passed as literal string to `ServiceNowClient.__init__()`. Python argparse handles quoting; user must quote the password correctly on the shell: `--password 'p@$$w0rd!'`.  
**Handled**: ✅ Yes — single quotes around password on CLI.

### EC-13: Concurrent Audit Runs
**Scenario**: Two auditors run the tool simultaneously, writing to the same output file.  
**Expected Behavior**: Last write wins. No file locking.  
**Handled**: ⚠️ Not handled — documented as R10 (output overwrite) in risk_report.md.  
**Mitigation**: Use timestamped output filenames.

### EC-14: ServiceNow API Version Change
**Scenario**: ServiceNow upgrades REST API version (e.g., `/api/now/v2/table/task`).  
**Expected Behavior**: Current endpoint `/api/now/table/task` returns 404. Exception caught → empty results.  
**Handled**: ⚠️ Not handled gracefully — silent zero-task report.  
**Mitigation**: Monitor ServiceNow API deprecation notices. Update endpoint URL if needed.

## Edge Case Coverage Matrix

| Edge Case | Category | Severity | Handled | Test Coverage | Risk |
|-----------|----------|----------|---------|---------------|------|
| EC-01 | Zero results | Low | ✅ | TS-10 | None |
| EC-02 | Single result | Low | ✅ | Implicit | None |
| EC-03 | Limit boundary | Medium | ✅ | None | Truncation risk |
| EC-04 | All null fields | Medium | ✅ | TS-03 | None |
| EC-05 | Malformed state | Medium | ✅ | TS-06 | None |
| EC-06 | Network timeout | High | ✅ | None | Silent failure |
| EC-07 | Non-JSON response | High | ✅ | None | Silent failure |
| EC-08 | Bot sys_id | Low | ✅ | None | None |
| EC-09 | DST transition | Low | ✅ | None | None |
| EC-10 | Duplicate tasks | Low | ⚠️ | None | Low |
| EC-11 | Long task number | Low | ✅ | None | None |
| EC-12 | Special chars password | Medium | ✅ | None | None |
| EC-13 | Concurrent runs | Medium | ⚠️ | None | Data loss |
| EC-14 | API version change | High | ⚠️ | None | Silent failure |

## Unhandled Edge Cases (Tracking)

| ID | Description | Priority | Target Release |
|----|-------------|----------|---------------|
| EC-06/07 | Add error logging to distinguish "no tasks" from "API failure" | P1 | v0.2.0 |
| EC-13 | Add file locking or timestamp default for output | P2 | v0.2.0 |
| EC-14 | Add API version negotiation (try v2, fallback to v1) | P2 | v0.3.0 |
