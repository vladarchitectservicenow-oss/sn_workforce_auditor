# SN Workforce Auditor — Architecture Summary

## Product Overview

SN Workforce Auditor is a Python-based forensic tool that audits ServiceNow's Autonomous Workforce resolution claims. ServiceNow's AI agents (Now Assist, AI Agent Studio) claim high resolution rates — typically 90%+. The Auditor independently verifies these claims by fetching actual task data via REST API and cross-referencing closure states, resolution attribution, and reassignment patterns to compute the *true* resolution rate.

## Tech Stack

| Component | Technology | Version / Notes |
|-----------|-----------|-----------------|
| Runtime | Python | 3.10+ (stdlib-only, zero external dependencies) |
| HTTP Client | `urllib.request` | Standard library — no pip install required |
| Data Models | `dataclasses` | Strongly-typed audit report structures |
| CLI | `argparse` | 5 arguments: instance, user, password, window-days, output |
| Serialization | `json` | Audit reports emitted as JSON |
| Testing | `unittest` + `unittest.mock` | Self-contained mocks; no external test deps |
| Target Platform | ServiceNow | Australia release (Xanadu compatible) |

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI (argparse)                        │
│  --instance  --user  --password  --window-days  --output    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    WorkforceAuditor                          │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  fetch_tasks()    │  │ parse_task() │  │ audit()      │  │
│  │  REST API → raw[] │→ │ raw → Task   │→ │ report gen   │  │
│  └──────────────────┘  └──────────────┘  └──────────────┘  │
│         │                                      │             │
│         ▼                                      ▼             │
│  ┌──────────────────┐              ┌──────────────────────┐ │
│  │ ServiceNowClient │              │ classify_task()      │ │
│  │ get_json()       │              │ 4 discrepancy types: │ │
│  │ Basic Auth        │              │ · unattributed       │ │
│  │ urllib.request     │              │ · claimed_cancelled  │ │
│  └──────────────────┘              │ · reassigned_resolved│ │
│                                     │ · frequent_reassign   │ │
│                                     └──────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      AuditReport                             │
│  Fields: date, window, totals, claimed_rate, actual_rate,   │
│          variance, discrepancies[]                           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                   audit_report.json
```

## Data Flow

1. **CLI invocation** → parses `--instance`, `--user`, `--password`, `--window-days`, `--output`
2. **ServiceNowClient** constructed with instance URL + Basic Auth header
3. **WorkforceAuditor.audit()** computes window: `now - window_days` to `now`
4. **fetch_tasks()** → `GET /api/now/table/task?sysparm_query=opened_at>=...&sysparm_fields=...&sysparm_limit=1000`
5. **parse_task()** → maps each raw dict to `Task` dataclass (sys_id, number, state, resolved_by, opened_at, closed_at, reassignment_count)
6. **classify_task()** → per-task discrepancy detection:
   - `state=resolved + no resolved_by` → `unattributed_resolution` (Moderate)
   - `state=cancelled + has resolved_by` → `claimed_cancelled` (Moderate)
   - `state=resolved + reassignment_count>0` → `reassigned_then_resolved` (Minor)
   - `state=resolved + reassignment_count>=2` → `frequent_reassignment` (Critical)
7. **AuditReport** computed: `claimed_rate = (claimed_resolved/total)*100`, `actual_rate = (actual_resolved/total)*100`, `variance = claimed - actual`
8. **report_to_dict()** → JSON serialization
9. **Output** → `audit_report.json` with full discrepancy list

## Discrepancy Classification Matrix

| Category | Trigger | Severity | Implication |
|----------|---------|----------|-------------|
| `unattributed_resolution` | Task resolved but `resolved_by` is null | Moderate | Bot may claim credit without attribution |
| `claimed_cancelled` | Task cancelled but `resolved_by` present | Moderate | Bot claimed resolution on cancelled work |
| `reassigned_then_resolved` | Resolved after 1+ reassignments | Minor | Human may have touched the task |
| `frequent_reassignment` | Resolved after 2+ reassignments | Critical | Strong evidence of human intervention |

## State Normalization Table

| State Code (SN) | Normalized | Meaning |
|----------------|------------|---------|
| 6 | resolved | Task resolved |
| 3 | closed | Task closed |
| 7 | closed | Task closed (alternate) |
| 4 | cancelled | Task cancelled |
| 8 | cancelled | Task cancelled (alternate) |

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Max tasks per audit | 1,000 (sysparm_limit) |
| HTTP timeout | 30 seconds |
| Typical audit window | 30 days |
| Memory profile | <50 MB for 1,000 tasks |
| Dependencies | Zero (stdlib only) |
| API calls per audit | 1 (single GET) |

## Security Model

- **Read-only API access**: The tool only fetches task data via GET; no writes, no state changes
- **Basic Auth**: Instance credentials passed via HTTP Basic Auth header
- **No credential persistence**: Credentials are CLI arguments only, never written to disk
- **No network egress** beyond the target ServiceNow instance
- **AGPL-3.0 license**: Full source transparency for audit reproducibility

## Edge Cases Handled

- Empty task results (no tasks in window) → report with all-zero metrics
- Network errors during API fetch → caught, returns empty list
- Missing optional fields (resolved_by, closed_at) → parsed as None
- Empty reassignment_count strings → parsed as 0
- Unknown state codes → pass-through (not classified)

## Compatibility

- ServiceNow Utah, Vancouver, Washington DC, Xanadu (Australia)
- REST API `/api/now/table/task` endpoint (available on all instances)
- Python 3.10+ (dataclasses, type hints)
