# SN Workforce Auditor — Execution Plan

## Phase Overview

| Phase | Name | Duration | Owner | Deliverables |
|-------|------|----------|-------|-------------|
| Phase 1 | Discovery & Planning | 1 day | Technical Lead | Architecture summary, dependency report, risk register, execution plan |
| Phase 2 | Core Development | 3 days | Backend Engineer | `src/auditor.py`, data models, classification engine |
| Phase 3 | Validation Suite | 2 days | QA Engineer | Test suite (10+ scenarios), regression cases, edge cases, validation checklist |
| Phase 4 | Documentation | 2 days | Technical Writer | README (2000+ words), SOP, API reference |
| Phase 5 | CI/CD & License | 1 day | DevOps | Git init, LICENSE, GitHub push, remote verification |
| Phase 6 | Audit Pilot | 2 days | Auditor | Run on test PDI (dev362840), validate real results |
| Phase 7 | Release | 1 day | Product Owner | Public GitHub release, marketing materials |

**Total: 12 days** for end-to-end delivery.

## Phase 1: Discovery & Planning (COMPLETE ✅)

- [x] architecture_summary.md — Component diagram, data flow, performance characteristics
- [x] dependency_report.md — Zero runtime deps, ServiceNow REST API contract, required permissions
- [x] risk_report.md — 12 risks scored (L×I), heatmap, mitigations
- [x] execution_plan.md — This document

## Phase 2: Core Development (COMPLETE ✅)

Tasks:
1. [x] Implement `ServiceNowClient` — urllib.request HTTP client with Basic Auth
2. [x] Implement `Task` dataclass — strong typing for task records
3. [x] Implement `Discrepancy` dataclass — classification metadata
4. [x] Implement `AuditReport` dataclass — structured audit output
5. [x] Implement `WorkforceAuditor.audit()` — fetch → parse → classify → aggregate
6. [x] Implement `report_to_dict()` — JSON serialization
7. [x] Implement `normalize_state()` — SN state code → human-readable mapping
8. [x] Implement `classify_task()` — 4 discrepancy detection rules
9. [x] Implement CLI (`argparse`) — 5 arguments
10. [x] Implement `main()` — entry point with JSON output

## Phase 3: Validation Suite (COMPLETE ✅)

Tasks:
1. [x] test_suite_SOP.md — 10+ test scenarios (negative cases included)
2. [x] regression_cases.md — Historical regression catalog
3. [x] edge_cases.md — Boundary conditions and corner cases
4. [x] validation_checklist.md — Pre-release quality gate
5. [x] test_auditor.py — 7 unittest cases (client, parsing, normalization, classification, audit report)

## Phase 4: Documentation (COMPLETE ✅)

Tasks:
1. [x] README.md — 2000+ words with Mermaid diagrams, ROI, troubleshooting
2. [x] SOP.md — Standard operating procedure for auditors
3. [x] LICENSE — AGPL-3.0 full text with copyright header

## Phase 5: CI/CD & License ✅

Actions:
1. [x] LICENSE contains "Copyright (C) 2026 Vladimir Kapustin"
2. [x] Git init + commit all files
3. [x] Push to `https://github.com/vladarchitectservicenow-oss/sn_workforce_auditor.git`
4. [x] Remote verification: README word count, LICENSE copyright, doc existence

## Phase 6: Audit Pilot

Plan for PDI testing:
1. Wake PDI `dev362840.service-now.com`
2. Run: `python3 src/auditor.py --instance dev362840 --user admin --password '7%%gXJzImsW7' --window-days 30 --output pilot_report.json`
3. Verify:
   - Report contains `total_tasks > 0`
   - `claimed_rate` and `actual_rate` are computed
   - Discrepancies are categorized correctly
   - JSON is valid and parsable
4. Cross-reference manual spot-check of 5 tasks

## Phase 7: Release

- [ ] Tag GitHub release v1.0.0
- [ ] Publish marketing materials (LinkedIn post, product description)
- [ ] Notify Australian ServiceNow community

## Execution Rules

1. **No hardcoded credentials in source** — all credentials via CLI args
2. **AGPL-3.0 header on every source file** — Copyright (c) 2026 Vladimir Kapustin
3. **Tests must pass before push** — `python3 -m unittest tests/test_auditor.py`
4. **Read-only API access only** — never modify production data
5. **Zero external dependencies** — stdlib Python only
6. **Russian docs internally, English on GitHub** — README and public surface in English

## Rollback Plan

If audit returns unexpected results:
1. Re-run with `--window-days 1` to isolate timeframe
2. Compare raw API response vs parsed data
3. Check `normalize_state()` mapping against actual instance state codes
4. If systemic issue found → file GitHub issue, revert to previous commit
