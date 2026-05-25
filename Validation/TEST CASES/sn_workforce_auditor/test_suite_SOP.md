# Test Suite SOP — SN Workforce Auditor

## Purpose

This document defines the complete test plan for SN Workforce Auditor. Every scenario must pass before a release is approved. The test suite validates correctness of task parsing, state normalization, discrepancy classification, audit report computation, and CLI behavior.

## Test Environment

- Python 3.10+
- No external dependencies (stdlib `unittest` + `unittest.mock`)
- No ServiceNow instance required (all tests self-contained with mocks)

## Run Command

```bash
python3 -m unittest tests/test_auditor.py -v
```

Total: **14 test cases** across 5 test classes.

## Test Scenarios

### TS-01: ServiceNowClient GET Request Parsing
**Purpose**: Verify that `get_json()` correctly parses a valid JSON response from the ServiceNow REST API.

**Input**: Mock response `{"result": [{"sys_id": "abc"}]}`  
**Expected**: `result["result"][0]["sys_id"] == "abc"`  
**Negative case**: N/A (happy path only)

### TS-02: Task Parsing — All Fields Present
**Purpose**: Verify `parse_task()` correctly maps all 7 fields from a raw API dict to a `Task` dataclass.

**Input**: Complete raw task dict with sys_id, number, state="6", resolved_by="bot_user", opened_at, closed_at, reassignment_count="3"  
**Expected**: `task.sys_id == "abc123"`, `task.resolved_by == "bot_user"`, `task.reassignment_count == 3`

### TS-03: Task Parsing — Missing Optional Fields
**Purpose**: Verify `parse_task()` handles null/missing optional fields gracefully.

**Input**: Raw dict with resolved_by=None, closed_at=None, reassignment_count=""  
**Expected**: `task.resolved_by is None`, `task.closed_at is None`, `task.reassignment_count == 0`  

### TS-04: State Normalization — Resolved/Closed States
**Purpose**: Verify `normalize_state()` maps ServiceNow state codes to human-readable labels.

**Input**: State codes "6", "3", "7"  
**Expected**: "resolved", "closed", "closed"

### TS-05: State Normalization — Cancelled States
**Purpose**: Verify cancelled state codes are recognized.

**Input**: State codes "4", "8"  
**Expected**: "cancelled", "cancelled"

### TS-06: State Normalization — Unknown State Pass-Through
**Purpose**: Verify unknown state codes are not transformed (no false classification).

**Input**: State code "new" (not in mapping)  
**Expected**: "new" (unchanged)

### TS-07: Discrepancy — Unattributed Resolution
**Purpose**: Verify detection when a task is resolved but has no `resolved_by`.

**Input**: Task with state="6" (resolved), resolved_by=None, reassignment_count=0  
**Expected**: 1 discrepancy, category="unattributed_resolution", severity="Moderate"

### TS-08: Discrepancy — Claimed Cancelled
**Purpose**: Verify detection when a cancelled task still has a resolution claim.

**Input**: Task with state="4" (cancelled), resolved_by="bot", reassignment_count=0  
**Expected**: Discrepancy with category="claimed_cancelled" present in results

### TS-09: Discrepancy — Critical Frequent Reassignment
**Purpose**: Verify detection when a task was reassigned 2+ times before resolution (strong human intervention signal).

**Input**: Task with state="6" (resolved), resolved_by="bot", reassignment_count=2  
**Expected**: Discrepancy with category="frequent_reassignment", severity="Critical"

### TS-10: Audit Report — Empty Tasks
**Purpose**: Verify audit handles zero tasks gracefully (empty API response).

**Input**: Mock client returning `{"result": []}`, window_days=7  
**Expected**: `report.total_tasks == 0`, `report.claimed_rate == 0.0`, `report.actual_rate == 0.0`

### TS-11: Audit Report — Rate Computation
**Purpose**: Verify the audit rate computation logic with mixed real data.

**Input**: 2 tasks — one resolved by bot (state=6, resolved_by="bot"), one closed without resolution (state=3, resolved_by=None)  
**Expected**: `total_tasks==2`, `claimed_resolved==2` (both are resolved-or-closed), `actual_resolved==1` (only the one with resolved_by), `claimed_rate≈100.0`, `actual_rate≈50.0`, `variance≈50.0`

### TS-12: Report to Dict Serialization
**Purpose**: Verify `report_to_dict()` correctly serializes an `AuditReport` including discrepancies.

**Input**: Report with 5 tasks, 1 discrepancy  
**Expected**: `dict["total_tasks"] == 5`, `len(dict["discrepancies"]) == 1`, `dict["variance"] == 40.0`

### TS-13: Reassigned Then Resolved (Minor)
**Purpose**: Verify the `reassigned_then_resolved` discrepancy is flagged for resolved tasks with 1 reassignment.

**Input**: Task with state="6", resolved_by="bot", reassignment_count=1  
**Expected**: Discrepancy with category="reassigned_then_resolved", severity="Minor"

### TS-14: Multiple Discrepancies Per Task
**Purpose**: Verify that a single task can trigger multiple discrepancy categories.

**Input**: Task with state="6", resolved_by="bot", reassignment_count=2  
**Expected**: Both `reassigned_then_resolved` (Minor, from reassignment_count>0) AND `frequent_reassignment` (Critical, from reassignment_count>=2) detected

## Test Execution Matrix

| Scenario | Type | Class | Status |
|----------|------|-------|--------|
| TS-01 | Happy Path | TestServiceNowClient | ✅ PASS |
| TS-02 | Happy Path | TestTaskParsing | ✅ PASS |
| TS-03 | Negative Case | TestTaskParsing | ✅ PASS |
| TS-04 | Happy Path | TestStateNormalization | ✅ PASS |
| TS-05 | Happy Path | TestStateNormalization | ✅ PASS |
| TS-06 | Edge Case | TestStateNormalization | ✅ PASS |
| TS-07 | Discrepancy | TestClassification | ✅ PASS |
| TS-08 | Discrepancy | TestClassification | ✅ PASS |
| TS-09 | Discrepancy | TestClassification | ✅ PASS |
| TS-10 | Edge Case | TestAuditReport | ✅ PASS |
| TS-11 | Integration | TestAuditReport | ✅ PASS |
| TS-12 | Serialization | TestAuditReport | ✅ PASS |
| TS-13 | Discrepancy | (pending add) | ⬜ TODO |
| TS-14 | Discrepancy (multi) | (pending add) | ⬜ TODO |

## Acceptance Gates

| Gate | Criterion | Required |
|------|-----------|----------|
| G0 | ≥10 scenarios defined (including negative) | ✅ (14 scenarios) |
| G1 | All implemented tests pass | ✅ |
| G2 | 100% branch coverage on `classify_task()` | ⬜ |
| G3 | 100% branch coverage on `normalize_state()` | ✅ |
| G4 | No external network calls during test execution | ✅ |
| G5 | Tests run in < 5 seconds | ✅ |

## Maintenance

When ServiceNow releases new state codes or changes field names:
1. Add new test case to `TestStateNormalization`
2. Update `normalize_state()` mapping
3. Add regression test in `regression_cases.md`
4. Bump test suite version
