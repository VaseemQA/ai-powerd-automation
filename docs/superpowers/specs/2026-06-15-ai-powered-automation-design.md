# AI-Powered Automation Framework — Design Spec (Prompt-Only)

**Date:** 2026-06-15
**Status:** Approved (revised — prompt-only architecture)

---

## Overview

A test automation framework with **zero application code**. The entire system is:

1. Test cases written as `.md` files (TMP regression format)
2. A `userdetails.md` file holding test user credentials
3. A GitHub Actions workflow that runs Claude Code CLI with a master prompt and Playwright MCP

Claude Code reads the `.md` test files, interprets the table-based steps, drives a headless Chromium browser via Playwright MCP, and reports results to the GitHub Actions summary and Slack — all from a single prompt invocation. No Node.js runner, no parser, no glue code.

**Target application:** The Money Platform (TMP)
**Environments:** UAT and INT (configurable per run)

---

## Repository Structure

```
ai-powerd-automation/
├── .github/
│   └── workflows/
│       ├── run-tests.yml          # Auto trigger on push/PR (UAT)
│       └── manual-run.yml         # Manual trigger — choose environment + test path
├── test-cases/
│   ├── repayment-plan/
│   │   └── admin-borrower-repayment-plan.md
│   └── ...                        # Organized by feature/module
├── config/
│   └── userdetails.md             # Test user credentials (one place, referenced by tests)
├── reports/
│   └── .gitkeep                   # Workflow uploads artifacts here
├── .mcp.json                      # Playwright MCP config (sanitized — no API keys)
└── README.md
```

**No** `package.json`, `scripts/`, `node_modules/`, or any source files. The repo is documentation + config + a workflow.

---

## How a Test Run Works

```
GitHub Actions trigger (push/PR or manual)
        ↓
Workflow installs Claude Code CLI + Playwright (headless Chromium)
        ↓
Workflow injects env vars: ANTHROPIC_API_KEY, TMP_UAT_URL / TMP_INT_URL,
SLACK_WEBHOOK_URL, ENVIRONMENT, TEST_PATH
        ↓
Workflow runs: `claude -p "<MASTER PROMPT>" --mcp-config .mcp.json --allowedTools "mcp__playwright__*" --output-format json`
        ↓
Claude reads:
  - config/userdetails.md (resolves credentials referenced by tests)
  - Test files at $TEST_PATH (one or more .md files in TMP regression format)
        ↓
For each test case, Claude executes table-based steps using Playwright MCP tools
(browser_navigate, browser_click, browser_fill_form, browser_snapshot, etc.)
        ↓
Claude evaluates Pass/Fail per the inline criteria in each step
        ↓
Claude writes a markdown summary report to $GITHUB_STEP_SUMMARY
        ↓
Workflow step posts a Slack message via $SLACK_WEBHOOK_URL with pass/fail counts + run URL
```

**Environment switching:** `TMP_UAT_URL` and `TMP_INT_URL` are stored as GitHub Secrets. The workflow exports the right one as `TMP_BASE_URL` based on the selected environment, and the master prompt instructs Claude to use it.

---

## Test Case Format (TMP Regression Style)

Each `.md` file is a self-contained regression document. Example structure:

```markdown
# Admin - Repayment Plan Setup Regression Test Cases

## Document Information
- **Product**: TMP Admin - Repayment Plan Management
- **Test Environment**: UAT (uat.themoneyplatform.com)
- **Last Updated**: 2026-03-13
- **Test Type**: Regression Testing

## Test Data Configuration

### Admin Credentials
See `config/userdetails.md` (admin user)

### Loan Parameters
- **Application ID**: 13243
- **Loan Amount**: £999.00
- **Term**: 6 weeks

---

## 1. Admin can setup repayment plan

### TC-RP-001: Admin Login to UAT
**Objective**: Verify admin can log into TMP admin panel
**Priority**: Critical
**Preconditions**: Valid admin credentials

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error |
| 2 | Enter admin email from userdetails | Email field populated |
| 3 | Enter admin password from userdetails | Password field populated (masked) |
| 4 | Click "Login" button | Redirect to `/admin/dashboard` |

**Pass Criteria**: Admin logged in and dashboard accessible
**Fail Criteria**: Login fails or 500 ISE error
```

**Format rules:**
- Document Information section at the top (metadata)
- Test Data Configuration section may reference `config/userdetails.md`
- One file may contain multiple `TC-XXX-NNN` test cases grouped under `## N. Title` sections
- Each test case has Objective, Priority, Preconditions, a step table, Pass/Fail criteria
- Step table uses three columns: `Step | Action | Expected Result`
- Inline `**FAIL** if ...` clauses inside the Expected Result column define hard-fail conditions Claude must check at every step
- Relative URLs (e.g., `/login`) are prefixed with `$TMP_BASE_URL` at runtime

---

## User Credentials — `config/userdetails.md`

```markdown
# Test User Credentials

## Admin
- **Email**: `vaseem@simformsolutions.com`
- **Password**: `Test@123`

## Borrower (Standard)
- **Email**: `borrower@example.com`
- **Password**: `Test@123`
```

When a test step says *"Enter admin email from userdetails"*, Claude resolves the value by reading this file. Adding a new user role = adding a new `## Role` section.

---

## GitHub Actions Workflows

### `run-tests.yml` — Auto trigger
- **Trigger:** push to `main`, pull_request to `main`
- **Environment:** UAT (default for auto)
- **Test scope:** all `.md` files under `test-cases/`
- **Steps:** Checkout → Install Node + Playwright browsers → Install Claude Code CLI → Run master prompt → Post Slack message → Upload `reports/` artifacts

### `manual-run.yml` — Manual trigger
- **Trigger:** `workflow_dispatch` from the GitHub Actions UI
- **Inputs:**
  - `environment`: dropdown — `UAT` | `INT`
  - `test_path`: text — file or directory (default `test-cases/`)
- **Steps:** identical to `run-tests.yml`, but uses the selected environment and test path

### Master Prompt (lives in both workflow files)

```
You are a TMP regression test execution agent. You have Playwright MCP browser tools available.

Inputs (from environment variables):
- TMP_BASE_URL: base URL for the target environment
- TEST_PATH: a file or directory under test-cases/

Steps:
1. Read config/userdetails.md to learn available test users.
2. Discover all *.md files under TEST_PATH (or use it directly if it's a file).
3. For each test case (each ## TC-... heading) in each file:
   a. Read the step table.
   b. Execute each step using Playwright MCP tools. Prefix relative URLs with TMP_BASE_URL.
   c. Resolve "from userdetails" references using config/userdetails.md.
   d. After each step, evaluate the Expected Result. Treat any "FAIL if ..." clause as a hard-fail condition (especially "FAIL if 500 ISE error").
   e. Take a screenshot on failure.
4. After all tests, write a markdown summary to the file at $GITHUB_STEP_SUMMARY:
   - Heading "# TMP Regression Test Run"
   - Environment, total/passed/failed counts
   - A results table: TC ID | Title | Status | Failed Step (if any) | Notes
5. Write a one-line summary to stdout: "PASS: X, FAIL: Y" so the workflow can parse it for Slack.

If anything is ambiguous, fail the affected test case with a clear reason rather than guessing.
```

The Slack notification is a separate workflow step that reads the stdout summary and the run URL, then posts to the webhook.

---

## GitHub Secrets Required

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude Code CLI authentication |
| `SLACK_WEBHOOK_URL` | Slack pass/fail notifications |
| `TMP_UAT_URL` | Base URL for UAT |
| `TMP_INT_URL` | Base URL for INT |

No password secret — credentials live in `config/userdetails.md` (per user preference; repo is private).

---

## Architecture Decisions

| Decision | Choice | Reason |
|---|---|---|
| Runner | Claude Code CLI (`claude -p`) — no custom code | User wants prompt-only architecture |
| Test interpretation | Pure prompting — no parser | Format is human-readable; Claude handles variation natively |
| Credentials | Single `config/userdetails.md` | DRY; one place to update across all tests |
| Master prompt location | Inline in workflow YAML | Single source of truth per workflow; matches "no scripts" intent |
| Browser | Headless Chromium via Playwright MCP | Already configured; standard for CI |
| Reporting | GitHub Actions summary + Slack | Zero-setup visibility for the team |
| Environment selection | GitHub Secrets + workflow input | URLs out of code; both envs from one workflow |
| `.mcp.json` | Sanitized: only Playwright MCP | Prevent committing API keys |
