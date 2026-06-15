# AI-Powered Automation Framework — Design Spec

**Date:** 2026-06-15  
**Status:** Approved

---

## Overview

An AI-powered test automation framework where Claude (via the Anthropic API) acts as the runtime brain. It reads plain-English test cases from `.md` files, interprets the steps, and drives a headless Chromium browser via Playwright MCP. Tests run automatically on push/PR and manually via GitHub Actions workflow dispatch. Results are published to the GitHub Actions summary and a Slack channel.

**Target application:** The Money Platform (TMP)  
**Environments:** UAT and INT (configurable per run)

---

## Repository Structure

```
ai-powerd-automation/
├── .github/
│   └── workflows/
│       ├── run-tests.yml          # Auto trigger on push/PR to main (UAT)
│       └── manual-run.yml         # Manual trigger — select environment + test path
├── test-cases/
│   ├── login/
│   │   └── login-basic.md
│   ├── dashboard/
│   │   └── dashboard-load.md
│   └── ...                        # Organized by feature/module
├── config/
│   └── userdetails.json           # Test user credentials (committed to repo)
├── scripts/
│   └── run-test.js                # Runner: reads .md → calls Claude API → drives Playwright MCP
├── reports/
│   └── .gitkeep
├── .mcp.json                      # Playwright MCP config (sanitized — no API keys)
├── package.json
├── .env.example
└── README.md
```

---

## Data Flow

```
GitHub Actions trigger (push/PR or manual)
        ↓
Select environment (UAT or INT) + test file(s)
        ↓
scripts/run-test.js starts
        ↓
1. Reads .md test case (plain English steps)
2. Reads config/userdetails.json for credentials
        ↓
3. Sends steps + credentials context to Claude API (claude-sonnet-4-6)
   Prompt: "You are a test automation agent. Execute these steps using Playwright MCP tools."
        ↓
4. Claude interprets steps + calls Playwright MCP tools
   (browser_navigate, browser_click, browser_fill_form, browser_snapshot, etc.)
        ↓
5. Playwright MCP drives headless Chromium on the GitHub Actions runner
        ↓
6. Pass/Fail result + screenshots collected
        ↓
7. Results posted to:
   - GitHub Actions job summary (markdown table)
   - Slack channel (webhook — pass/fail count + link to run)
```

**Environment switching:** Base URL (`TMP_UAT_URL` or `TMP_INT_URL`) is injected from GitHub Secrets into the Claude prompt. The same `.md` test file runs on both environments unchanged.

---

## Test Case Format

Each `.md` file follows this structure:

```markdown
---
name: Login - Valid Credentials
environment: both
url_path: /login
---

## Steps

1. Navigate to the login page
2. Enter the standard user email from userdetails
3. Enter the standard user password from userdetails
4. Click the "Sign In" button
5. Verify the dashboard page loads successfully
6. Take a screenshot

## Expected Result
User should be logged in and redirected to the dashboard.
```

**Rules:**
- Frontmatter holds metadata: test name, target environments, starting URL path
- Steps are plain English — no code, no CSS selectors
- Credentials referenced as "standard user from userdetails" or "admin user from userdetails" — Claude reads `config/userdetails.json`
- `Expected Result` is the assertion Claude evaluates at the end
- Screenshots captured on failure automatically; on success when step says "Take a screenshot"

---

## User Credentials

**`config/userdetails.json`** — committed to repo:

```json
{
  "users": {
    "standard": {
      "email": "testuser@tmp.com",
      "password": "yourpassword"
    },
    "admin": {
      "email": "admin@tmp.com",
      "password": "adminpassword"
    }
  }
}
```

---

## GitHub Actions Workflows

### `run-tests.yml` — Auto trigger
- **Trigger:** Push to `main`, PR to `main`
- **Environment:** UAT (default)
- **Test scope:** All `.md` files in `test-cases/`
- **Steps:** Checkout → Node.js setup → Install deps → Install Playwright browsers → Run all tests → Post GitHub summary → Slack notification

### `manual-run.yml` — Manual trigger
- **Trigger:** `workflow_dispatch` from GitHub Actions UI
- **Inputs:**
  - `environment`: dropdown — `UAT` | `INT`
  - `test_path`: text — e.g. `test-cases/login/login-basic.md` or `test-cases/` for all
- **Steps:** Same as auto trigger but uses selected environment + test path

---

## GitHub Secrets Required

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude API calls at runtime |
| `SLACK_WEBHOOK_URL` | Slack pass/fail notifications |
| `TMP_UAT_URL` | Base URL for UAT environment |
| `TMP_INT_URL` | Base URL for INT environment |

---

## Architecture Decisions

| Decision | Choice | Reason |
|---|---|---|
| Claude role | Runtime interpreter | Maximum flexibility — edit `.md` = change test, no recompilation |
| Claude model | `claude-sonnet-4-6` | Best balance of speed and capability for browser automation |
| Browser | Headless Chromium via Playwright MCP | Already configured in `.mcp.json` |
| Credentials storage | `config/userdetails.json` committed to repo | User preference; repo is private |
| Results | GitHub summary + Slack | Zero-setup reporting + team visibility |
| Environment selection | GitHub Secrets + workflow input | Keeps URLs out of code, supports both envs from one workflow |
