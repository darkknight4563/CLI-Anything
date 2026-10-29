# cli-anything-gatekeeper — Test Plan & Results

## Part 1 — Plan

### Inventory
- `test_core.py` — ~30 unit tests, no network.
- `test_full_e2e.py` — ~20 E2E tests: a local mock of the Gatekeeper API + Supabase
  (full happy path, auth, quota, saved reports), CLI subprocess tests, and **live**
  tests against https://www.gatekeeper.blog.

### Unit plan
| Module | What | Edge cases |
|---|---|---|
| `core/extract.py` | DOCX paragraphs/tabs, XLSX `[Sheet N]` TSV with shared + inline strings, PDF text, TXT | wrong ext, >10 MB, >10 files, corrupt zip, scanned-PDF warning, empty file, folder expansion |
| `core/evaluate.py` | `parse_weights` | defaults → `None`, sum ≠ 100, unknown criterion, bad number, out of range |
| `core/evaluate.py` | `build_payload` | unreadable file rejected |
| `core/session.py` | JWT claims, save/load 0600, env override, refresh when near expiry | corrupt file, no refresh token |
| `core/report.py` | load (3 shapes), summary, compliance filters, pricing discrepancies, risks, markdown | missing sections, non-report JSON |
| `utils/gk_backend.py` | magic-link fragment parsing | no tokens in link |

### E2E plan
1. **Tender evaluation workflow** — simulates a buyer evaluating 2 bids against an RFP:
   extract folder (DOCX RFP, PDF bids, XLSX BOM, TXT) → sign in with code → `evaluate -o`
   → `report summary/compliance --status FAIL --mandatory/pricing/risks/markdown`.
   Verified: payload sent equals extracted text, weights sent only when customised,
   Bearer token present, report saved and re-readable offline, Markdown contains the ranking.
2. **Plan wall** — 402 from `/api/evaluate` → exit code 3 with upgrade URL.
3. **Expired session** — token within 120 s of expiry is refreshed before the call.
4. **History** — `reports list` / `reports get -o` query `evaluations` with the user's email + RLS token.
5. **Magic link** — `auth complete --link` follows `/auth/v1/verify` without redirect and reads tokens.
6. **Live service** — `health` returns 200 from production; `/api/config` exposes Supabase URL;
   `evaluate` with an invalid token returns exit 2 (401, no credit spent); invalid email rejected by
   real Supabase auth. A real paid evaluation runs only when `GATEKEEPER_LIVE_TOKEN` is set.

## Part 2 — Results

Run 2026-10-29, Python 3.13, `CLI_ANYTHING_FORCE_INSTALLED=1 pytest -v --tb=no`
(live tests hit https://www.gatekeeper.blog):

```
test_core.py::TestExtract::test_docx_paragraphs_and_tabs PASSED [  1%]
test_core.py::TestExtract::test_xlsx_layout_matches_web PASSED [  3%]
test_core.py::TestExtract::test_pdf_text PASSED [  5%]
test_core.py::TestExtract::test_txt PASSED [  7%]
test_core.py::TestExtract::test_scanned_pdf_warning PASSED [  8%]
test_core.py::TestExtract::test_rejects_wrong_type PASSED [ 10%]
test_core.py::TestExtract::test_rejects_large_file PASSED [ 12%]
test_core.py::TestExtract::test_missing_file PASSED [ 14%]
test_core.py::TestExtract::test_corrupt_docx_becomes_warning PASSED [ 16%]
test_core.py::TestExtract::test_empty_txt_warns PASSED [ 17%]
test_core.py::TestExtract::test_folder_expansion_skips_unaccepted PASSED [ 19%]
test_core.py::TestExtract::test_max_files PASSED [ 21%]
test_core.py::TestExtract::test_no_inputs PASSED [ 23%]
test_core.py::TestExtract::test_summarize_has_no_text PASSED [ 25%]
test_core.py::TestWeights::test_none PASSED [ 26%]
test_core.py::TestWeights::test_defaults_return_none PASSED [ 28%]
test_core.py::TestWeights::test_custom PASSED [ 30%]
test_core.py::TestWeights::test_sum_must_be_100 PASSED [ 32%]
test_core.py::TestWeights::test_unknown_criterion PASSED [ 33%]
test_core.py::TestWeights::test_bad_number_and_range PASSED [ 35%]
test_core.py::TestWeights::test_payload_rejects_unreadable PASSED [ 37%]
test_core.py::TestWeights::test_payload_shape PASSED [ 39%]
test_core.py::TestSession::test_jwt_claims PASSED [ 41%]
test_core.py::TestSession::test_save_load_permissions PASSED [ 42%]
test_core.py::TestSession::test_env_overrides PASSED [ 44%]
test_core.py::TestSession::test_not_signed_in PASSED [ 46%]
test_core.py::TestSession::test_refresh_near_expiry PASSED [ 48%]
test_core.py::TestSession::test_expired_without_refresh PASSED [ 50%]
test_core.py::TestSession::test_corrupt_session_file PASSED [ 51%]
test_core.py::TestSession::test_clear PASSED [ 53%]
test_core.py::test_fragment_tokens PASSED  [ 55%]
test_core.py::TestReport::test_load_shapes PASSED [ 57%]
test_core.py::TestReport::test_summary_sorted_and_counts PASSED [ 58%]
test_core.py::TestReport::test_compliance_filters PASSED [ 60%]
test_core.py::TestReport::test_pricing_discrepancies PASSED [ 62%]
test_core.py::TestReport::test_risks_include_missing_docs PASSED [ 64%]
test_core.py::TestReport::test_markdown PASSED [ 66%]
test_core.py::TestReport::test_empty_report PASSED [ 67%]
test_full_e2e.py::TestTenderWorkflow::test_full_evaluation_pipeline PASSED [ 69%]
test_full_e2e.py::TestTenderWorkflow::test_default_weights_not_sent PASSED [ 71%]
test_full_e2e.py::TestTenderWorkflow::test_human_output PASSED [ 73%]
test_full_e2e.py::TestErrors::test_plan_wall_exit_3 PASSED [ 75%]
test_full_e2e.py::TestErrors::test_not_signed_in_exit_2 PASSED [ 76%]
test_full_e2e.py::TestErrors::test_bad_token_exit_2 PASSED [ 78%]
test_full_e2e.py::TestErrors::test_bad_weights_exit_1_no_call PASSED [ 80%]
test_full_e2e.py::TestErrors::test_wrong_code PASSED [ 82%]
test_full_e2e.py::TestErrors::test_dry_run_sends_nothing PASSED [ 83%]
test_full_e2e.py::TestSessionLifecycle::test_magic_link_and_refresh PASSED [ 85%]
test_full_e2e.py::TestSessionLifecycle::test_status_verify_logout PASSED [ 87%]
test_full_e2e.py::TestSavedReports::test_list_and_get PASSED [ 89%]
test_full_e2e.py::TestCLIBasics::test_help_and_version PASSED [ 91%]
test_full_e2e.py::TestCLIBasics::test_real_excel_blank_styled_cells PASSED [ 92%]
test_full_e2e.py::TestLive::test_live_health PASSED [ 94%]
test_full_e2e.py::TestLive::test_live_invalid_token_is_401 PASSED [ 96%]
test_full_e2e.py::TestLive::test_live_supabase_auth_reachable PASSED [ 98%]
test_full_e2e.py::TestLive::test_live_real_evaluation SKIPPED [100%]
======================== 55 passed, 1 skipped in 12.59s ========================
```

### Summary
- 55 passed, 1 skipped (the credit-consuming live evaluation, gated on `GATEKEEPER_LIVE_TOKEN`), ~12 s.
- Live: production `/api/health` 200; `/api/evaluate` rejects an invalid token with 401 → exit 2;
  production Supabase auth validates the email format.

### Coverage notes
- The success path of `/api/evaluate` is exercised against a contract-faithful mock (request and
  response shapes taken from the production `evaluate.html`). A live evaluation needs a signed-in
  account and spends a credit — run `GATEKEEPER_LIVE_TOKEN=… pytest -k live_real`.
- The magic-link email itself is not tested end to end (would send real mail).
- Finding while testing: the web app's XLSX parser merges formatted blank cells into the next
  cell, shifting BOM columns. The CLI keeps the column (`test_real_excel_blank_styled_cells`).
