# SN Workforce Auditor — Dependency Report

## Runtime Dependencies

SN Workforce Auditor is designed as a **zero-dependency** Python application. All imports come from the Python standard library.

| Import | Library | Version Requirement | Purpose |
|--------|---------|-------------------|---------|
| `argparse` | stdlib | Any | CLI argument parsing |
| `json` | stdlib | Any | JSON serialization of audit reports |
| `sys` | stdlib | Any | System-level utilities |
| `dataclasses` | stdlib | Python 3.7+ | Strongly-typed data models |
| `datetime` | stdlib | Any | Time window computation |
| `typing` | stdlib | Python 3.5+ | Type annotations (List, Optional) |
| `urllib.parse` | stdlib | Any | URL joining |
| `urllib.request` | stdlib | Any | HTTP client (Basic Auth, GET requests) |

## Python Runtime Dependency

| Dependency | Minimum Version | Rationale |
|-----------|----------------|-----------|
| Python | 3.10 | Uses `dataclasses` with `Optional[...]` typing, f-string formatting |

No `pip install` required. No `requirements.txt`. No `pyproject.toml` needed for runtime.

## External Service Dependencies

| Service | Endpoint | Authentication | Required? | Purpose |
|---------|----------|---------------|-----------|---------|
| ServiceNow Instance | `/api/now/table/task` | Basic Auth (user:password) | Yes | Fetch task records for audit window |
| ServiceNow Instance | DNS Resolution | N/A | Yes | Instance hostname must be resolvable |

## ServiceNow REST API Contract

### Required Table Access

| Table | Access Level | Purpose |
|-------|-------------|---------|
| `task` | Read-only | Primary data source — parent table for incident, change, request, etc. |

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `sys_id` | String | Unique record identifier |
| `number` | String | Display number (e.g., INC0001001) |
| `state` | String | State code (3, 4, 6, 7, 8) |
| `resolved_by` | Reference / String | Resolver user or bot |
| `opened_at` | DateTime | Task creation timestamp |
| `closed_at` | DateTime | Task closure timestamp |
| `reassignment_count` | Integer | Number of times task was reassigned |

### Required User Permissions

| Permission | Role | Purpose |
|-----------|------|---------|
| `task_read` | Any role with read access to `task` table | Fetch task records |
| `REST API access` | `rest_api` role or equivalent | Access `/api/now/table/*` endpoints |

## Test Dependencies

| Dependency | Purpose |
|-----------|---------|
| `unittest` (stdlib) | Test framework |
| `unittest.mock` (stdlib) | Mock ServiceNowClient for offline testing |

No external test frameworks required.

## System Dependencies (CI/CD)

| Tool | Version | Purpose |
|------|---------|---------|
| `git` | Any | Version control |
| `python3` | 3.10+ | Test execution |
| `curl` | Any | GitHub API verification (optional, post-deploy only) |

## Network Dependencies

| Endpoint | Protocol | Port | TLS | Purpose |
|----------|----------|------|-----|---------|
| `*.service-now.com` | HTTPS | 443 | Required | ServiceNow REST API calls |
| `github.com` | HTTPS | 443 | Required | Git push operations |

## Compatibility Matrix

| ServiceNow Version | Compatible |
|-------------------|-----------|
| Utah | ✅ |
| Vancouver | ✅ |
| Washington DC | ✅ |
| Xanadu | ✅ |
| Australia | ✅ (target) |
