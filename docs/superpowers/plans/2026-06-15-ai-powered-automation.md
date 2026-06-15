# AI-Powered Automation Framework Implementation Plan (Prompt-Only)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a zero-code regression test framework. Test cases live as `.md` files in TMP regression format, credentials in a `userdetails.md` file, and GitHub Actions workflows run Claude Code CLI with a master prompt that drives Playwright MCP to execute the tests and report to GitHub summary + Slack.

**Architecture:** No application code — only `.md` test cases, a single `userdetails.md`, two GitHub Actions workflows (auto + manual), a sanitized `.mcp.json`, and an updated `README.md`. Each workflow runs `claude -p "<master prompt>" --mcp-config .mcp.json --allowedTools "mcp__playwright__*"` with environment variables for env/test path/secrets, and Claude does discovery, parsing, execution, and reporting via prompting.

**Tech Stack:** GitHub Actions, Claude Code CLI, Playwright MCP (`@playwright/mcp`), headless Chromium. No Node project, no test frameworks.

---

## File Structure

**Files to create:**
- `config/userdetails.md` — test user credentials (Admin, Borrower)
- `test-cases/repayment-plan/admin-borrower-repayment-plan.md` — sample test file in TMP regression format (the one the user shared)
- `.github/workflows/run-tests.yml` — auto trigger on push/PR (UAT)
- `.github/workflows/manual-run.yml` — manual dispatch with environment + test path inputs
- `reports/.gitkeep` — placeholder for workflow artifact uploads

**Files to modify:**
- `.mcp.json` — strip the hardcoded Airtable API keys (keep only Playwright MCP)
- `.gitignore` — append a single rule to keep `reports/` clean except `.gitkeep`
- `README.md` — replace with usage instructions

**Cleanup:**
- Delete `.mcp.json:Zone.Identifier` (Windows alternate-stream artifact, not needed)

---

## Task 1: Sanitize `.mcp.json` and Clean Up

**Files:**
- Modify: `.mcp.json`
- Delete: `.mcp.json:Zone.Identifier`

The current `.mcp.json` (untracked) contains hardcoded Airtable API keys. Strip everything but Playwright MCP before committing.

- [ ] **Step 1: Replace `.mcp.json` with the sanitized version**

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest", "--headless"]
    }
  },
  "globalSettings": {
    "timeout": 30000,
    "retryAttempts": 3,
    "logLevel": "info"
  }
}
```

- [ ] **Step 2: Delete the Zone.Identifier file**

```bash
rm /home/vaseem/ai-powerd-automation/.mcp.json:Zone.Identifier
```

- [ ] **Step 3: Verify no API keys remain anywhere in the file**

```bash
grep -i "patffCGn\|api_key\|API_KEY\|airtable" /home/vaseem/ai-powerd-automation/.mcp.json && echo FAIL || echo OK
```

Expected: `OK` (grep finds nothing).

- [ ] **Step 4: Commit**

```bash
git add .mcp.json
git commit -m "chore: sanitize .mcp.json — keep only Playwright MCP"
```

---

## Task 2: Create `config/userdetails.md`

**Files:**
- Create: `config/userdetails.md`

- [ ] **Step 1: Create the file**

Path: `/home/vaseem/ai-powerd-automation/config/userdetails.md`

Contents:

```markdown
# Test User Credentials

Single source of truth for test users. Test cases reference this file by user role
(e.g. "Enter the admin email from userdetails").

## Admin
- **Email**: `vaseem@simformsolutions.com`
- **Password**: `Test@123`

## Borrower (Standard)
- **Email**: `borrower@example.com`
- **Password**: `Test@123`
```

(Replace the borrower placeholders with real values when you have them.)

- [ ] **Step 2: Commit**

```bash
git add config/userdetails.md
git commit -m "feat: add userdetails.md for test credentials"
```

---

## Task 3: Create the Sample Test File

**Files:**
- Create: `test-cases/repayment-plan/admin-borrower-repayment-plan.md`

This is the TMP repayment-plan regression suite the user shared, with only one adjustment: hardcoded `https://uat.themoneyplatform.com/...` URLs are replaced with relative paths so the same file runs on both UAT and INT.

- [ ] **Step 1: Create the directory**

```bash
mkdir -p /home/vaseem/ai-powerd-automation/test-cases/repayment-plan
```

- [ ] **Step 2: Create the file**

Path: `/home/vaseem/ai-powerd-automation/test-cases/repayment-plan/admin-borrower-repayment-plan.md`

Contents (full):

````markdown
# Admin - Repayment Plan Setup Regression Test Cases

## Document Information
- **Product**: TMP Admin - Repayment Plan Management
- **Test Environment**: UAT or INT (selected at run time)
- **Last Updated**: 2026-03-13
- **Test Type**: Regression Testing - Admin Repayment Plan Setup
- **User Role**: Admin / Super Admin

---

## Test Data Configuration

### Admin Credentials
See `config/userdetails.md` (Admin user).

### Loan Parameters (Test Application)
- **Application ID**: 13243
- **Loan ID**: 3058
- **Borrower**: Accept Test
- **Loan Amount**: £999.00
- **Term**: 6 weeks
- **Daily Rate**: 0.8%
- **Installments**: 2

### Repayment Plan Parameters
- **Amount**: £999.00
- **Installment Amount**: £200.00
- **Frequency**: Monthly
- **Placement Type**: Last Weekday
- **Start Date**: 30/03/2026

> All URLs in step tables are relative paths. The runner prefixes them with the
> selected environment's base URL (UAT or INT).

---

## 1. Admin can setup repayment plan for a specific loan

### TC-RP-001: Admin Login
**Objective**: Verify admin can successfully login to TMP admin panel
**Priority**: Critical
**Preconditions**: Valid admin credentials

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed with Email, Password fields and Login button. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated (masked). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to admin dashboard `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed, Today's Payments section visible. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in and dashboard accessible
**Fail Criteria**: Login fails, dashboard not accessible, or 500 ISE error on any step

---

### TC-RP-002: Admin can setup repayment plan for a specific loan
**Objective**: Verify admin can setup repayment plan for a specific loan
**Priority**: High
**Preconditions**: Admin logged in (from TC-RP-001)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/loans/list/1/started` | Loan page displayed with list of started loans. **FAIL** if 500 ISE error is displayed |
| 2 | Select any one loan from the list and click "Details" | Loan details page displayed with loan summary. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "loaned"). **FAIL** if 500 ISE error is displayed |
| 4 | Note the "Amount to be repaid £XYZ" from the loan summary panel | Loan summary with amount displayed. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Repayment Plan" button | Repayment plan setup form displayed. **FAIL** if 500 ISE error is displayed |
| 6 | Enter the Amount as £XYZ (from step 4) | Amount field populated with loan amount. **FAIL** if 500 ISE error is displayed |
| 7 | Enter an installment amount less than £XYZ | Installment amount field populated. **FAIL** if 500 ISE error is displayed |
| 8 | Select frequency "Monthly" | Monthly frequency selected. **FAIL** if 500 ISE error is displayed |
| 9 | Select placement type "Last Weekday" | Last Weekday placement type selected. **FAIL** if 500 ISE error is displayed |
| 10 | Verify application state | State badge displayed (e.g., "planned"). **FAIL** if 500 ISE error is displayed |
| 11 | Click "Accept Test" borrower name link, then click "Impersonate this user" | Redirected to borrower dashboard as impersonated user. **FAIL** if 500 ISE error is displayed |
| 12 | Verify "You are currently on a Repayment Plan." text | Repayment plan confirmation text visible. **FAIL** if 500 ISE error is displayed |
| 13 | Navigate to `/logout` | User logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Repayment plan setup by admin works without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed

---

## 2. Borrower can setup repayment plan for a specific loan

### TC-RP-003: Admin Login (for impersonation)
**Objective**: Admin login as a precondition to impersonate a borrower
**Priority**: Critical

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in
**Fail Criteria**: Login fails or 500 ISE error

---

### TC-RP-004: Borrower can setup repayment plan for a specific loan
**Objective**: Verify borrower can setup repayment plan for a specific loan
**Priority**: High
**Preconditions**: Admin logged in (TC-RP-003)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/loans/list/1/started` | Loan page displayed with started loans. **FAIL** if 500 ISE error is displayed |
| 2 | Select a loan and click "Details" | Loan details page displayed. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "loaned"). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Accept Test" borrower name link | User details page displayed. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Toggle User Flag" | User flag toggled successfully. **FAIL** if 500 ISE error is displayed |
| 6 | Click "Impersonate this user" | Redirected to borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 7 | Click "Help with my loan" | Help options displayed. **FAIL** if button missing or 500 ISE |
| 8 | Click "Enter a repayment plan" | Repayment plan setup form displayed. **FAIL** if button missing or 500 ISE |
| 9 | Enter installment amount less than total balance | Amount field populated. **FAIL** if 500 ISE error is displayed |
| 10 | Select frequency "Monthly" | Monthly frequency selected. **FAIL** if 500 ISE error is displayed |
| 11 | Select placement type "Last Weekday" | Last Weekday placement type selected. **FAIL** if 500 ISE error is displayed |
| 12 | Click "Continue" | Repayment plan summary/confirmation page displayed. **FAIL** if 500 ISE error is displayed |
| 13 | Tick the "I agree for my payments to be processed... CPA" checkbox | Checkbox checked. **FAIL** if 500 ISE error is displayed |
| 14 | Click "Confirm Repayment Plan" | Redirected to card payment page. **FAIL** if 500 ISE error is displayed |
| 15 | Enter card details: number `4477 0000 0000 0006`, expiry `12/99`, CVV `111`, name `setup complete` | Card details entered and payment processed. **FAIL** if 500 ISE error is displayed |
| 16 | Refresh the page two times and click "Continue" | Page refreshed and continue completed. **FAIL** if 500 ISE error is displayed |
| 17 | Verify "You are currently on a Repayment Plan." text | Confirmation text visible. **FAIL** if 500 ISE error is displayed |
| 18 | Navigate to `/logout` | Logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Borrower-initiated repayment plan setup works without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed

---

## 3. Borrower can pay repayment plan installment

### TC-RP-005: Admin Login (for impersonation)
**Objective**: Admin login as a precondition to impersonate a borrower
**Priority**: Critical

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in
**Fail Criteria**: Login fails or 500 ISE error

---

### TC-RP-006: Borrower can pay repayment plan installment
**Objective**: Verify borrower can pay a repayment plan installment
**Priority**: High
**Preconditions**: Admin logged in (TC-RP-005)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/repayment-plans` | Repayment plans page with list of plans displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Select a repayment plan and click "Details" | Plan details page displayed. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "planned"). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Accept Test" borrower name link, then "Toggle User Flag" | User flag toggled. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Impersonate this user" | Redirected to borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 6 | Verify "You are currently on a Repayment Plan." text | Text visible on borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 7 | Verify "Make a Payment" button is visible and click it | Payment page displayed with amount + payment method options. **FAIL** if button missing or 500 ISE |
| 8 | Enter installment amount (e.g., £40) and complete the payment | Payment processed via card/bank. **FAIL** if 500 ISE error is displayed |
| 9 | Verify "Your payment was successful" text on dashboard | Success message visible. **FAIL** if 500 ISE error is displayed |
| 10 | Navigate to `/logout` | Logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Borrower can pay repayment plan installment without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed
````

- [ ] **Step 3: Commit**

```bash
git add test-cases/
git commit -m "feat: add TMP repayment-plan regression test suite"
```

---

## Task 4: Create the Master Prompt as a Reusable Block

The master prompt is identical across both workflows. Storing it as a heredoc inside each workflow file is fine — but to keep them in sync, we paste the same prompt verbatim in both workflow files. Below is the canonical text used in Tasks 5 and 6.

**Canonical master prompt (copy this exactly into both workflows in Tasks 5 and 6):**

````
You are a TMP regression test execution agent. You have Playwright MCP browser tools available (mcp__playwright__*).

Inputs (from environment variables):
- TMP_BASE_URL: base URL for the target environment (e.g. https://uat.themoneyplatform.com)
- TEST_PATH: a file path or directory under test-cases/

Your job:
1. Read config/userdetails.md to learn available test users (Admin, Borrower, etc).
2. Discover all *.md files at $TEST_PATH (or treat $TEST_PATH as a single file if it ends in .md). Process them in alphabetical order.
3. For each test case (each "### TC-..." heading) in each file:
   a. Read the step table (columns: Step, Action, Expected Result).
   b. Execute each step using Playwright MCP browser tools.
      - Prefix any relative URL like "/login" with $TMP_BASE_URL.
      - When a step says "from userdetails", look up the value in config/userdetails.md by the role mentioned in the step (Admin or Borrower).
      - Take a screenshot when a step explicitly says to.
   c. After each step, evaluate the Expected Result. Treat any "FAIL if ..." clause inside the Expected Result column as a hard-fail condition. The "FAIL if 500 ISE error is displayed" check must run on every step.
   d. On any step failure: take a screenshot, record the failed step number and a one-line reason, and stop running the remaining steps in that test case. Continue with the next test case.
4. After all test cases finish, write the following to the file path in $GITHUB_STEP_SUMMARY (use the Bash tool with `cat >> "$GITHUB_STEP_SUMMARY"` or equivalent):

# TMP Regression Test Run

**Environment:** $TMP_BASE_URL
**Test Path:** $TEST_PATH
**Total:** N  **Passed:** P  **Failed:** F

| TC ID | Title | Status | Failed Step | Notes |
| --- | --- | --- | --- | --- |
| TC-RP-001 | Admin Login | ✅ pass | — | — |
| TC-RP-002 | ... | ❌ fail | 5 | Repayment Plan button missing |

5. Print exactly one line to stdout in this format so the workflow can parse it for Slack:
   RESULT_SUMMARY: passed=P failed=F total=N

If anything is genuinely ambiguous (e.g. a referenced selector cannot be located after a reasonable attempt), fail the affected test case with a clear one-line reason rather than guessing.
````

This task has no commit on its own — it documents the prompt for Tasks 5 and 6.

- [ ] **Step 1: Acknowledge the canonical prompt above will be embedded in both workflow files in Tasks 5 and 6.**

(No file changes, no commit. Move to Task 5.)

---

## Task 5: Auto-Trigger Workflow

**Files:**
- Create: `.github/workflows/run-tests.yml`

- [ ] **Step 1: Create the directory**

```bash
mkdir -p /home/vaseem/ai-powerd-automation/.github/workflows
```

- [ ] **Step 2: Create the workflow file**

Path: `/home/vaseem/ai-powerd-automation/.github/workflows/run-tests.yml`

Contents:

```yaml
name: Run Tests (auto)

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    env:
      ENVIRONMENT: UAT
      TEST_PATH: test-cases/
      TMP_BASE_URL: ${{ secrets.TMP_UAT_URL }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Playwright browsers
        run: npx -y playwright install --with-deps chromium

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Run regression suite
        id: run
        run: |
          set -o pipefail
          claude -p "$(cat <<'PROMPT'
          You are a TMP regression test execution agent. You have Playwright MCP browser tools available (mcp__playwright__*).

          Inputs (from environment variables):
          - TMP_BASE_URL: base URL for the target environment (e.g. https://uat.themoneyplatform.com)
          - TEST_PATH: a file path or directory under test-cases/

          Your job:
          1. Read config/userdetails.md to learn available test users (Admin, Borrower, etc).
          2. Discover all *.md files at $TEST_PATH (or treat $TEST_PATH as a single file if it ends in .md). Process them in alphabetical order.
          3. For each test case (each "### TC-..." heading) in each file:
             a. Read the step table (columns: Step, Action, Expected Result).
             b. Execute each step using Playwright MCP browser tools.
                - Prefix any relative URL like "/login" with $TMP_BASE_URL.
                - When a step says "from userdetails", look up the value in config/userdetails.md by the role mentioned in the step (Admin or Borrower).
                - Take a screenshot when a step explicitly says to.
             c. After each step, evaluate the Expected Result. Treat any "FAIL if ..." clause inside the Expected Result column as a hard-fail condition. The "FAIL if 500 ISE error is displayed" check must run on every step.
             d. On any step failure: take a screenshot, record the failed step number and a one-line reason, and stop running the remaining steps in that test case. Continue with the next test case.
          4. After all test cases finish, append the following to the file path in $GITHUB_STEP_SUMMARY (use the Bash tool with cat >> "$GITHUB_STEP_SUMMARY"):

          # TMP Regression Test Run

          **Environment:** $TMP_BASE_URL
          **Test Path:** $TEST_PATH
          **Total:** N  **Passed:** P  **Failed:** F

          | TC ID | Title | Status | Failed Step | Notes |
          | --- | --- | --- | --- | --- |

          (Fill the table with one row per test case using ✅ pass or ❌ fail.)

          5. Print exactly one line to stdout in this format so the workflow can parse it for Slack:
             RESULT_SUMMARY: passed=P failed=F total=N

          If anything is genuinely ambiguous, fail the affected test case with a one-line reason rather than guessing.
          PROMPT
          )" \
            --mcp-config .mcp.json \
            --allowedTools "mcp__playwright__*,Bash,Read,Glob" \
            --output-format text \
            | tee claude-output.txt

          # Extract result summary line for Slack step
          summary_line="$(grep -E '^RESULT_SUMMARY:' claude-output.txt | tail -n1 || true)"
          if [ -z "$summary_line" ]; then
            summary_line="RESULT_SUMMARY: passed=0 failed=0 total=0"
          fi
          echo "summary=$summary_line" >> "$GITHUB_OUTPUT"
          # Fail the job if any test failed
          failed=$(echo "$summary_line" | sed -n 's/.*failed=\([0-9]\+\).*/\1/p')
          if [ "${failed:-0}" -gt 0 ]; then exit 1; fi

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: |
            reports/
            claude-output.txt

      - name: Notify Slack
        if: always()
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SUMMARY: ${{ steps.run.outputs.summary }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          ENV_NAME: UAT
        run: |
          if [ -z "$SLACK_WEBHOOK_URL" ]; then
            echo "SLACK_WEBHOOK_URL not set; skipping Slack notification"
            exit 0
          fi
          icon=":white_check_mark:"
          case "$SUMMARY" in *failed=0*) ;; *) icon=":x:" ;; esac
          payload=$(printf '{"text":"%s TMP regression on %s — %s\\nRun: %s"}' \
            "$icon" "$ENV_NAME" "$SUMMARY" "$RUN_URL")
          curl -sS -X POST -H 'Content-Type: application/json' \
            --data "$payload" "$SLACK_WEBHOOK_URL"
```

- [ ] **Step 3: Lint check (offline)**

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('/home/vaseem/ai-powerd-automation/.github/workflows/run-tests.yml')); print('YAML OK')"
```

Expected: `YAML OK`.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/run-tests.yml
git commit -m "ci: add auto-trigger workflow for push and PR"
```

---

## Task 6: Manual-Trigger Workflow

**Files:**
- Create: `.github/workflows/manual-run.yml`

- [ ] **Step 1: Create the workflow file**

Path: `/home/vaseem/ai-powerd-automation/.github/workflows/manual-run.yml`

Contents:

```yaml
name: Run Tests (manual)

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'UAT'
        type: choice
        options:
          - UAT
          - INT
      test_path:
        description: 'Test file or directory (e.g. test-cases/repayment-plan/admin-borrower-repayment-plan.md)'
        required: true
        default: 'test-cases/'
        type: string

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    env:
      ENVIRONMENT: ${{ inputs.environment }}
      TEST_PATH: ${{ inputs.test_path }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Resolve base URL for environment
        id: env
        run: |
          if [ "$ENVIRONMENT" = "INT" ]; then
            echo "base_url=${{ secrets.TMP_INT_URL }}" >> "$GITHUB_OUTPUT"
          else
            echo "base_url=${{ secrets.TMP_UAT_URL }}" >> "$GITHUB_OUTPUT"
          fi

      - name: Install Playwright browsers
        run: npx -y playwright install --with-deps chromium

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Run regression suite
        id: run
        env:
          TMP_BASE_URL: ${{ steps.env.outputs.base_url }}
        run: |
          set -o pipefail
          claude -p "$(cat <<'PROMPT'
          You are a TMP regression test execution agent. You have Playwright MCP browser tools available (mcp__playwright__*).

          Inputs (from environment variables):
          - TMP_BASE_URL: base URL for the target environment
          - TEST_PATH: a file path or directory under test-cases/

          Your job:
          1. Read config/userdetails.md to learn available test users (Admin, Borrower, etc).
          2. Discover all *.md files at $TEST_PATH (or treat $TEST_PATH as a single file if it ends in .md). Process them in alphabetical order.
          3. For each test case (each "### TC-..." heading) in each file:
             a. Read the step table (columns: Step, Action, Expected Result).
             b. Execute each step using Playwright MCP browser tools.
                - Prefix any relative URL like "/login" with $TMP_BASE_URL.
                - When a step says "from userdetails", look up the value in config/userdetails.md by the role mentioned in the step (Admin or Borrower).
                - Take a screenshot when a step explicitly says to.
             c. After each step, evaluate the Expected Result. Treat any "FAIL if ..." clause inside the Expected Result column as a hard-fail condition. The "FAIL if 500 ISE error is displayed" check must run on every step.
             d. On any step failure: take a screenshot, record the failed step number and a one-line reason, and stop running the remaining steps in that test case. Continue with the next test case.
          4. After all test cases finish, append the following to the file path in $GITHUB_STEP_SUMMARY (use the Bash tool with cat >> "$GITHUB_STEP_SUMMARY"):

          # TMP Regression Test Run

          **Environment:** $TMP_BASE_URL
          **Test Path:** $TEST_PATH
          **Total:** N  **Passed:** P  **Failed:** F

          | TC ID | Title | Status | Failed Step | Notes |
          | --- | --- | --- | --- | --- |

          (Fill the table with one row per test case using ✅ pass or ❌ fail.)

          5. Print exactly one line to stdout in this format so the workflow can parse it for Slack:
             RESULT_SUMMARY: passed=P failed=F total=N

          If anything is genuinely ambiguous, fail the affected test case with a one-line reason rather than guessing.
          PROMPT
          )" \
            --mcp-config .mcp.json \
            --allowedTools "mcp__playwright__*,Bash,Read,Glob" \
            --output-format text \
            | tee claude-output.txt

          summary_line="$(grep -E '^RESULT_SUMMARY:' claude-output.txt | tail -n1 || true)"
          if [ -z "$summary_line" ]; then
            summary_line="RESULT_SUMMARY: passed=0 failed=0 total=0"
          fi
          echo "summary=$summary_line" >> "$GITHUB_OUTPUT"
          failed=$(echo "$summary_line" | sed -n 's/.*failed=\([0-9]\+\).*/\1/p')
          if [ "${failed:-0}" -gt 0 ]; then exit 1; fi

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: |
            reports/
            claude-output.txt

      - name: Notify Slack
        if: always()
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SUMMARY: ${{ steps.run.outputs.summary }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          ENV_NAME: ${{ inputs.environment }}
        run: |
          if [ -z "$SLACK_WEBHOOK_URL" ]; then
            echo "SLACK_WEBHOOK_URL not set; skipping Slack notification"
            exit 0
          fi
          icon=":white_check_mark:"
          case "$SUMMARY" in *failed=0*) ;; *) icon=":x:" ;; esac
          payload=$(printf '{"text":"%s TMP regression on %s — %s\\nRun: %s"}' \
            "$icon" "$ENV_NAME" "$SUMMARY" "$RUN_URL")
          curl -sS -X POST -H 'Content-Type: application/json' \
            --data "$payload" "$SLACK_WEBHOOK_URL"
```

- [ ] **Step 2: Lint check**

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('/home/vaseem/ai-powerd-automation/.github/workflows/manual-run.yml')); print('YAML OK')"
```

Expected: `YAML OK`.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/manual-run.yml
git commit -m "ci: add manual workflow with environment selector"
```

---

## Task 7: `.gitignore` and Reports Placeholder

**Files:**
- Modify: `.gitignore`
- Create: `reports/.gitkeep`

- [ ] **Step 1: Append rule to `.gitignore`**

Append (only) these lines to the existing `.gitignore`:

```
# Workflow output
reports/*
!reports/.gitkeep
claude-output.txt
```

- [ ] **Step 2: Create reports placeholder**

```bash
mkdir -p /home/vaseem/ai-powerd-automation/reports && touch /home/vaseem/ai-powerd-automation/reports/.gitkeep
```

- [ ] **Step 3: Commit**

```bash
git add .gitignore reports/.gitkeep
git commit -m "chore: ignore workflow output, keep reports/ dir"
```

---

## Task 8: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace `README.md` contents**

```markdown
# ai-powerd-automation

Zero-code regression test framework for The Money Platform (TMP). Test cases live as
`.md` files. GitHub Actions runs Claude Code CLI with a master prompt, and Claude
drives a headless Chromium browser via Playwright MCP to execute the tests.
Results are posted to the GitHub Actions summary and a Slack channel.

## How a run works

1. You add or edit a `.md` test file under `test-cases/`.
2. GitHub Actions runs the test (auto on push/PR, or manual via the Actions tab).
3. The workflow runs `claude -p "<master prompt>" --mcp-config .mcp.json --allowedTools "mcp__playwright__*"`.
4. Claude reads `config/userdetails.md` and the test file(s), drives the browser, and writes a results table to the job summary.
5. A Slack webhook posts a one-line pass/fail summary with a link to the run.

## Project layout

- `test-cases/` — `.md` test files (TMP regression format)
- `config/userdetails.md` — single source of truth for test user credentials
- `.github/workflows/run-tests.yml` — auto on push/PR (UAT)
- `.github/workflows/manual-run.yml` — manual dispatch with environment + test path
- `.mcp.json` — Playwright MCP config

## Test file format

Use `## N. Section Title` to group cases, then one or more `### TC-XXX-NNN: Title`
test cases. Each test case has a step table with three columns: Step, Action,
Expected Result. Inline `**FAIL** if ...` clauses define hard-fail conditions
Claude must check at every step.

See `test-cases/repayment-plan/admin-borrower-repayment-plan.md` for a full example.

## Required GitHub secrets

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude Code authentication |
| `SLACK_WEBHOOK_URL` | Pass/fail notifications |
| `TMP_UAT_URL` | UAT base URL |
| `TMP_INT_URL` | INT base URL |

## Adding a new test

1. Create a new `.md` file under `test-cases/<feature>/`.
2. Write test cases using the TMP regression format (Document Information,
   Test Data Configuration, then `## N.` sections containing `### TC-...` cases).
3. If you need a new test user role, add a section to `config/userdetails.md`
   and reference it from steps as "from userdetails (Role)".
4. Push. The auto workflow runs against UAT. Use the manual workflow to target INT.

## Adding a new test user

Edit `config/userdetails.md` and add a new role section:

```markdown
## NewRole
- **Email**: `someone@example.com`
- **Password**: `secret`
```

Then reference it in test steps: "Enter the NewRole email from userdetails".
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README for prompt-only architecture"
```

---

## Task 9: End-to-End Smoke Verification (Manual Gate)

This is a manual gate before pushing. Confirms the framework is wired up correctly.

- [ ] **Step 1: Confirm GitHub Secrets are set in the repo**

Open the repo on GitHub → Settings → Secrets and variables → Actions, and confirm:
- `ANTHROPIC_API_KEY`
- `SLACK_WEBHOOK_URL`
- `TMP_UAT_URL`
- `TMP_INT_URL`

(These cannot be set via CLI without `gh` installed; do this in the browser.)

- [ ] **Step 2: Confirm `config/userdetails.md` has real credentials**

Open `/home/vaseem/ai-powerd-automation/config/userdetails.md` and verify Admin email/password are real (the borrower placeholder can stay until needed).

- [ ] **Step 3: Push the branch**

```bash
cd /home/vaseem/ai-powerd-automation
git push -u origin feat/ai-automation-framework
```

- [ ] **Step 4: Open a PR to `main` to trigger the auto workflow**

Open the GitHub UI, create a PR `feat/ai-automation-framework → main`. Watch the "Run Tests (auto)" check.
Expected: workflow runs, results table appears in the job summary, Slack message arrives.

- [ ] **Step 5: Trigger a manual run against INT**

GitHub Actions tab → "Run Tests (manual)" → Run workflow → environment: `INT`, test_path: `test-cases/repayment-plan/admin-borrower-repayment-plan.md`.
Expected: workflow runs against INT, results posted as above.

- [ ] **Step 6: Merge once green, or fix and re-run**

If both runs are green, merge. If something fails, examine the job summary and the `claude-output.txt` artifact.

---

## Self-Review Notes

**Spec coverage (point-by-point):**
- Repository structure (no scripts) → Tasks 1, 2, 3, 5, 6, 7
- TMP regression test format → Task 3 (sample) + master prompt in Tasks 5/6
- `userdetails.md` with role lookup → Task 2 + master-prompt step 3b
- Auto + manual workflows → Tasks 5, 6
- Master prompt embedded in workflows → Task 4 (canonical) + Tasks 5, 6 (embedded)
- GitHub Actions summary + Slack → workflow steps in Tasks 5, 6
- Required secrets table → README in Task 8 + workflow `env:` blocks
- Sanitized `.mcp.json` → Task 1
- Architecture decisions table reflects no-scripts choice → README in Task 8

**Placeholder scan:** No TBDs, no "implement later", no "similar to Task N" without code, no missing commit messages.

**Type/name consistency:** `TMP_BASE_URL` is the var name in both workflows and the master prompt. `TEST_PATH` likewise. `ENVIRONMENT` in input → resolves to `TMP_BASE_URL` in manual-run. `RESULT_SUMMARY:` prefix is identical in both workflows for Slack parsing.

**Security:** Task 1 strips Airtable keys from `.mcp.json` before any commit. `userdetails.md` is committed per user preference (private repo).
