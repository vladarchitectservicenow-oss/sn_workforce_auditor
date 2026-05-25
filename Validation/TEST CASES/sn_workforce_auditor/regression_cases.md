# SN Workforce Auditor — Regression Cases

## Purpose

This document catalogs historical bugs, known regressions, and verification scenarios that must be re-tested after every code change. Each case includes the original symptom, root cause, fix, and verification steps.

## Regression Cases

### RC-01: reassignment_count String-to-Int Parsing
**Discovered**: v0.1.0 initial development  
**Symptom**: `TypeError: '>' not supported between instances of 'str' and 'int'` when comparing `reassignment_count > 0`  
**Root Cause**: ServiceNow REST API returns `reassignment_count` as string `"3"`, not as int `3`. `parse_task()` did not cast.  
**Fix**: Added `int(raw.get("reassignment_count", 0) or 0)` — double fallback for empty strings.  
**Verification**: Feed raw dict with `reassignment_count: ""` → expect `task.reassignment_count == 0`. Feed `"5"` → expect `5`.

### RC-02: Empty resolved_by Causes False Classification
**Discovered**: v0.1.0 initial development  
**Symptom**: Tasks with `resolved_by: ""` were classified as `unattributed_resolution` but should pass through.  
**Root Cause**: `resolved_by` empty string is falsy in Python but not None. `or None` check correctly handled `""` → None.  
**Fix**: Already handled (`raw.get("resolved_by") or None`).  
**Verification**: Feed raw dict with `resolved_by: ""` → expect `task.resolved_by is None`.

### RC-03: State Code "7" Not Recognized as Closed
**Discovered**: v0.1.0 initial development  
**Symptom**: Tasks with state `"7"` were not counted in `claimed_resolved`.  
**Root Cause**: `normalize_state()` only mapped `"3"` → "closed". State `"7"` is also a closed state in ServiceNow.  
**Fix**: Added mapping `"7": "closed"`.  
**Verification**: Feed state `"7"` → expect `"closed"`. Feed state `"8"` → expect `"cancelled"`.

### RC-04: Zero Division on Empty Task Set
**Discovered**: v0.1.0 initial development  
**Symptom**: `ZeroDivisionError` when `audit()` returns zero tasks.  
**Root Cause**: `claimed_rate = (claimed_resolved / total * 100)` with `total = 0`.  
**Fix**: Added guard: `(claimed_resolved / total * 100) if total else 0.0`.  
**Verification**: Mock client returning `{"result": []}` → expect `claimed_rate == 0.0`, no crash.

### RC-05: Audit Report Timestamp Format
**Discovered**: v0.1.0 initial development  
**Symptom**: `audit_date` in report used `datetime.now()` without timezone, producing ambiguous timestamps.  
**Root Cause**: `datetime.now()` returns naive datetime.  
**Fix**: Changed to `datetime.now(timezone.utc)`.  
**Verification**: Parse `audit_date` from report JSON → expect ISO 8601 with timezone offset (e.g., `2024-01-15T00:00:00+00:00`).

### RC-06: urllib.request.urlopen Mock Signature
**Discovered**: v0.1.0 testing phase  
**Symptom**: `AttributeError: 'FakeResponse' object has no attribute '__enter__'` in tests.  
**Root Cause**: `urllib.request.urlopen()` is used as context manager (`with ... as resp:`). Mock must implement `__enter__` and `__exit__`.  
**Fix**: Added context manager protocol to `FakeResponse` class in tests.  
**Verification**: Run `test_get_json_parses_response` → expect `mock_urlopen.assert_called_once()` passes.

### RC-07: reassignment_count ≥ 2 Triggers Both Categories
**Discovered**: v0.1.0 classification logic review  
**Symptom**: Task with `reassignment_count=2` only triggered `frequent_reassignment` (Critical) but missed `reassigned_then_resolved` (Minor) since the check `reassignment_count > 0` also applies.  
**Root Cause**: Both checks are independent — this is correct behavior.  
**Fix**: No fix needed. Verified that both category checks fire.  
**Verification**: Create task with reassignment_count=2, resolved → expect 2 discrepancies (one Minor, one Critical).

### RC-08: Normalized State Case Sensitivity
**Discovered**: v0.1.0 state mapping review  
**Symptom**: If ServiceNow returns state "Resolved" (capitalized), `normalize_state()` returns it unchanged.  
**Root Cause**: Mapping uses exact string matching.  
**Fix**: `.lower()` applied to output: `return mapping.get(state, state).lower()`.  
**Verification**: Feed state "NEW" → expect "new". Feed "CLOSED" → expect "closed".

## Regression Test Execution

| Release | Date | RC-01 | RC-02 | RC-03 | RC-04 | RC-05 | RC-06 | RC-07 | RC-08 | Status |
|---------|------|-------|-------|-------|-------|-------|-------|-------|-------|--------|
| v0.1.0 | 2026-05-25 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ALL PASS |
