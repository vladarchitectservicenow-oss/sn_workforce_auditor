# SN Workforce Auditor — Validation Checklist

## Purpose

This checklist must be completed and signed off before each release. All items must pass. Any failure blocks the release.

## Release Information

| Field | Value |
|-------|-------|
| Release | v0.1.0 |
| Date | 2026-05-25 |
| Validator | Automated CI (Hermes cron pipeline) |
| Repository | `vladarchitectservicenow-oss/sn_workforce_auditor` |

## Code Quality Gates

| # | Check | Method | Threshold | Result |
|---|-------|--------|-----------|--------|
| CQ-01 | No syntax errors | `python3 -m py_compile src/auditor.py` | Exit 0 | ⬜ |
| CQ-02 | No hardcoded credentials | `grep -rPn 'DEFAULT_PASS\|password=' src/` | Empty output | ⬜ |
| CQ-03 | AGPL-3.0 header on source | `head -5 src/auditor.py` | Contains "AGPL-3.0" and "Vladimir Kapustin" | ⬜ |
| CQ-04 | Zero external dependencies | `grep -r '^import\|^from' src/auditor.py` | All imports from stdlib | ⬜ |
| CQ-05 | Proper type hints | Visual inspection of dataclasses and method signatures | All public methods annotated | ⬜ |
| CQ-06 | CLI has --help | `python3 src/auditor.py --help` | Exit 0, shows usage | ⬜ |

## Test Gates

| # | Check | Method | Threshold | Result |
|---|-------|--------|-----------|--------|
| T-01 | All unit tests pass | `python3 -m unittest tests/test_auditor.py -v` | Exit 0, no failures | ⬜ |
| T-02 | ≥10 test scenarios | Count test methods in `test_auditor.py` | ≥10 | ⬜ |
| T-03 | Negative cases covered | Review `test_suite_SOP.md` | ≥2 negative scenarios | ⬜ |
| T-04 | Regression cases documented | `regression_cases.md` | ≥8 numbered cases | ⬜ |
| T-05 | Edge cases documented | `edge_cases.md` | ≥10 documented edges | ⬜ |
| T-06 | No network in tests | `grep -r 'urllib.request.urlopen' tests/` | Only mocked (via `@patch`) | ⬜ |

## Documentation Gates

| # | Check | Method | Threshold | Result |
|---|-------|--------|-----------|--------|
| D-01 | README ≥ 2000 words | `wc -w README.md` | ≥2000 | ⬜ |
| D-02 | README has Mermaid diagram | `grep -c '```mermaid' README.md` | ≥1 | ⬜ |
| D-03 | README has ROI section | `grep -c 'ROI\|Return on Investment' README.md` | ≥1 | ⬜ |
| D-04 | README has Troubleshooting | `grep -c 'Troubleshooting' README.md` | ≥1 | ⬜ |
| D-05 | No duplicate sections | `grep -c '^## Overview$' README.md` | 1 | ⬜ |
| D-06 | README license matches LICENSE | Check header references AGPL-3.0 if LICENSE is AGPL-3.0 | Match | ⬜ |
| D-07 | Architecture summary exists | `wc -l memory/checkpoints/architecture_summary.md` | ≥40 lines | ⬜ |
| D-08 | Dependency report exists | `wc -l memory/checkpoints/dependency_report.md` | ≥30 lines | ⬜ |
| D-09 | Risk report exists | `grep -c 'P0\|P1\|P2\|P3' memory/checkpoints/risk_report.md` | ≥5 risk tags | ⬜ |
| D-10 | Execution plan exists | `wc -l memory/checkpoints/execution_plan.md` | ≥30 lines | ⬜ |

## License & Copyright Gates

| # | Check | Method | Threshold | Result |
|---|-------|--------|-----------|--------|
| L-01 | LICENSE file exists | `wc -l LICENSE` | ≥600 lines (AGPL-3.0 full text) | ⬜ |
| L-02 | Copyright in LICENSE | `grep -c 'Vladimir Kapustin' LICENSE` | ≥2 | ⬜ |
| L-03 | Copyright format correct | `grep 'Copyright.*2026.*Vladimir Kapustin' LICENSE` | Match (no abbreviations) | ⬜ |
| L-04 | Source file headers | `head -5 src/auditor.py tests/test_auditor.py SOP.md` | All have "Vladimir Kapustin" | ⬜ |

## Git & Push Gates

| # | Check | Method | Threshold | Result |
|---|-------|--------|-----------|--------|
| G-01 | .gitignore exists | `test -f .gitignore` | True | ⬜ |
| G-02 | .gitignore excludes __pycache__ | `grep '__pycache__/' .gitignore` | Match | ⬜ |
| G-03 | Git status clean | `git status --porcelain` | Empty (all staged) | ⬜ |
| G-04 | Git push succeeds | `git push` | Exit 0 | ⬜ |
| G-05 | Remote README ≥ 2000w | `curl raw/.../README.md \| wc -w` | ≥2000 | ⬜ |
| G-06 | Remote LICENSE copyright | `curl raw/.../LICENSE \| grep -c 'Vladimir Kapustin'` | ≥2 | ⬜ |
| G-07 | Remote Phase 1 docs exist | `curl api.github.com/repos/.../contents/memory/checkpoints/architecture_summary.md` | Has `sha` | ⬜ |
| G-08 | Remote Phase 2 docs exist | `curl api.github.com/repos/.../contents/Validation/TEST%20CASES/.../test_suite_SOP.md` | Has `sha` | ⬜ |

## Acceptance Criteria

| Gate | Required | Status |
|------|----------|--------|
| All CQ (Code Quality) gates | All PASS | ⬜ |
| All T (Test) gates | All PASS | ⬜ |
| All D (Documentation) gates | All PASS | ⬜ |
| All L (License) gates | All PASS | ⬜ |
| All G (Git & Push) gates | All PASS | ⬜ |

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Developer | Vladimir Kapustin | 2026-05-25 | |
| QA Lead | — | — | |
| Release Manager | — | — | |

**Release approved when all gates show PASS. Any FAIL blocks the release.**
