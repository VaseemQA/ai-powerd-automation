# AI-Powered Automation Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a test automation framework where Claude (Anthropic API) reads plain-English test cases from `.md` files at runtime and drives a headless Chromium browser via Playwright MCP, executed on GitHub Actions.

**Architecture:** A Node.js runner orchestrates four components: a test-case parser (markdown → structured steps), a credentials loader (`userdetails.json`), a Playwright MCP stdio client (subprocess + tool bridge), and a Claude agentic loop (sends steps + tool schemas, executes tool calls, evaluates `Expected Result`). Two GitHub workflows trigger the runner on push/PR (auto, UAT) and manual dispatch (UAT or INT, selectable test path). Reports flow to GitHub Actions summary and a Slack webhook.

**Tech Stack:** Node.js 20, `@anthropic-ai/sdk`, `@modelcontextprotocol/sdk`, `@playwright/mcp`, `gray-matter` (frontmatter parser), Node built-in `node:test` runner, GitHub Actions.

---

## File Structure

**Files to create:**
- `package.json` — Node project + deps
- `.env.example` — env var template (no secrets)
- `.mcp.json` — sanitized Playwright MCP config (committed)
- `.gitignore` — append `.env`, `reports/*`, `node_modules`
- `config/userdetails.json` — test user credentials
- `test-cases/login/login-basic.md` — sample test case
- `scripts/run-test.js` — entry point (discover tests, orchestrate, report)
- `scripts/lib/parseTestCase.js` — markdown → `{name, environment, urlPath, steps, expectedResult}`
- `scripts/lib/loadCredentials.js` — read & validate `config/userdetails.json`
- `scripts/lib/playwrightMcpClient.js` — start MCP subprocess, expose tool schemas + invoke
- `scripts/lib/runTestWithClaude.js` — Claude agentic loop: prompt → tool calls → result
- `scripts/lib/reporter.js` — GitHub summary markdown + Slack webhook payload
- `tests/parseTestCase.test.js` — unit tests
- `tests/loadCredentials.test.js` — unit tests
- `tests/reporter.test.js` — unit tests
- `.github/workflows/run-tests.yml` — auto trigger
- `.github/workflows/manual-run.yml` — manual dispatch
- `reports/.gitkeep` — placeholder for runner output dir

**Files to modify:**
- `README.md` — usage instructions
- `.mcp.json` — strip Airtable API keys (security)

---

## Task 1: Initialize Node Project

**Files:**
- Create: `package.json`
- Create: `.gitignore` (append rules)

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "ai-powerd-automation",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=20"
  },
  "scripts": {
    "test": "node --test tests/",
    "run-test": "node scripts/run-test.js"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.65.0",
    "@modelcontextprotocol/sdk": "^1.0.4",
    "@playwright/mcp": "^0.0.40",
    "gray-matter": "^4.0.3"
  }
}
```

- [ ] **Step 2: Append to `.gitignore`**

Add these lines to the existing `.gitignore`:

```
# Test runner output
reports/*
!reports/.gitkeep

# Local env file (never commit secrets)
.env
```

- [ ] **Step 3: Install deps and verify**

Run: `cd /home/vaseem/ai-powerd-automation && npm install`
Expected: Creates `node_modules/` and `package-lock.json`, no errors.

- [ ] **Step 4: Create reports directory placeholder**

Run: `mkdir -p /home/vaseem/ai-powerd-automation/reports && touch /home/vaseem/ai-powerd-automation/reports/.gitkeep`

- [ ] **Step 5: Commit**

```bash
git add package.json package-lock.json .gitignore reports/.gitkeep
git commit -m "chore: initialize Node project with deps"
```

---

## Task 2: Test Case Parser (TDD)

**Files:**
- Create: `tests/parseTestCase.test.js`
- Create: `scripts/lib/parseTestCase.js`

- [ ] **Step 1: Write the failing test**

Create `tests/parseTestCase.test.js`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { parseTestCase } from '../scripts/lib/parseTestCase.js';

test('parses frontmatter and steps from a valid test case', () => {
  const md = `---
name: Login - Valid Credentials
environment: both
url_path: /login
---

## Steps

1. Navigate to the login page
2. Enter the standard user email from userdetails
3. Click the "Sign In" button

## Expected Result
User should be logged in and redirected to the dashboard.
`;

  const result = parseTestCase(md);

  assert.equal(result.name, 'Login - Valid Credentials');
  assert.equal(result.environment, 'both');
  assert.equal(result.urlPath, '/login');
  assert.deepEqual(result.steps, [
    'Navigate to the login page',
    'Enter the standard user email from userdetails',
    'Click the "Sign In" button',
  ]);
  assert.equal(
    result.expectedResult,
    'User should be logged in and redirected to the dashboard.'
  );
});

test('throws when frontmatter is missing required fields', () => {
  const md = `---
name: Bad Test
---

## Steps
1. Do something

## Expected Result
Things happen.
`;
  assert.throws(() => parseTestCase(md), /url_path/);
});

test('throws when Steps section is missing', () => {
  const md = `---
name: Bad Test
environment: both
url_path: /x
---

## Expected Result
Nope.
`;
  assert.throws(() => parseTestCase(md), /Steps/);
});

test('throws when Expected Result section is missing', () => {
  const md = `---
name: Bad Test
environment: both
url_path: /x
---

## Steps
1. Do a thing
`;
  assert.throws(() => parseTestCase(md), /Expected Result/);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: FAIL — `Cannot find module '../scripts/lib/parseTestCase.js'`

- [ ] **Step 3: Write the implementation**

Create `scripts/lib/parseTestCase.js`:

```javascript
import matter from 'gray-matter';

export function parseTestCase(markdown) {
  const parsed = matter(markdown);
  const fm = parsed.data;

  if (!fm.name) throw new Error('Frontmatter missing required field: name');
  if (!fm.environment) throw new Error('Frontmatter missing required field: environment');
  if (!fm.url_path) throw new Error('Frontmatter missing required field: url_path');

  const body = parsed.content;
  const stepsMatch = body.match(/##\s+Steps\s*\n([\s\S]*?)(?=\n##\s+|$)/);
  if (!stepsMatch) throw new Error('Test case missing "## Steps" section');

  const steps = stepsMatch[1]
    .split('\n')
    .map((line) => line.match(/^\s*\d+\.\s+(.*\S)\s*$/))
    .filter(Boolean)
    .map((m) => m[1]);

  if (steps.length === 0) throw new Error('Test case has no numbered steps');

  const expectedMatch = body.match(/##\s+Expected Result\s*\n([\s\S]*?)(?=\n##\s+|$)/);
  if (!expectedMatch) throw new Error('Test case missing "## Expected Result" section');

  const expectedResult = expectedMatch[1].trim();

  return {
    name: fm.name,
    environment: fm.environment,
    urlPath: fm.url_path,
    steps,
    expectedResult,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: All 4 tests in `parseTestCase.test.js` PASS.

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/parseTestCase.js tests/parseTestCase.test.js
git commit -m "feat: add test case markdown parser"
```

---

## Task 3: Credentials Loader (TDD)

**Files:**
- Create: `tests/loadCredentials.test.js`
- Create: `scripts/lib/loadCredentials.js`
- Create: `config/userdetails.json`

- [ ] **Step 1: Create the credentials file**

Create `config/userdetails.json`:

```json
{
  "users": {
    "standard": {
      "email": "testuser@tmp.com",
      "password": "REPLACE_WITH_REAL_PASSWORD"
    },
    "admin": {
      "email": "admin@tmp.com",
      "password": "REPLACE_WITH_REAL_PASSWORD"
    }
  }
}
```

- [ ] **Step 2: Write the failing test**

Create `tests/loadCredentials.test.js`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { writeFileSync, mkdtempSync, rmSync } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';
import { loadCredentials } from '../scripts/lib/loadCredentials.js';

function makeTempFile(contents) {
  const dir = mkdtempSync(join(tmpdir(), 'creds-'));
  const path = join(dir, 'userdetails.json');
  writeFileSync(path, contents);
  return { path, cleanup: () => rmSync(dir, { recursive: true, force: true }) };
}

test('loads valid credentials file', () => {
  const { path, cleanup } = makeTempFile(JSON.stringify({
    users: {
      standard: { email: 'a@b.com', password: 'pw1' },
      admin: { email: 'c@d.com', password: 'pw2' },
    },
  }));
  try {
    const creds = loadCredentials(path);
    assert.equal(creds.users.standard.email, 'a@b.com');
    assert.equal(creds.users.admin.password, 'pw2');
  } finally {
    cleanup();
  }
});

test('throws when file is missing', () => {
  assert.throws(() => loadCredentials('/nonexistent/path.json'), /not found/i);
});

test('throws when JSON is malformed', () => {
  const { path, cleanup } = makeTempFile('{ not json');
  try {
    assert.throws(() => loadCredentials(path), /parse/i);
  } finally {
    cleanup();
  }
});

test('throws when users key is missing', () => {
  const { path, cleanup } = makeTempFile(JSON.stringify({ wrong: {} }));
  try {
    assert.throws(() => loadCredentials(path), /users/);
  } finally {
    cleanup();
  }
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: FAIL — `Cannot find module '../scripts/lib/loadCredentials.js'`

- [ ] **Step 4: Write the implementation**

Create `scripts/lib/loadCredentials.js`:

```javascript
import { readFileSync, existsSync } from 'node:fs';

export function loadCredentials(path) {
  if (!existsSync(path)) {
    throw new Error(`Credentials file not found: ${path}`);
  }
  const raw = readFileSync(path, 'utf8');
  let parsed;
  try {
    parsed = JSON.parse(raw);
  } catch (err) {
    throw new Error(`Failed to parse credentials JSON: ${err.message}`);
  }
  if (!parsed.users || typeof parsed.users !== 'object') {
    throw new Error('Credentials file must contain a "users" object');
  }
  return parsed;
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add scripts/lib/loadCredentials.js tests/loadCredentials.test.js config/userdetails.json
git commit -m "feat: add credentials loader and userdetails template"
```

---

## Task 4: Reporter (TDD)

**Files:**
- Create: `tests/reporter.test.js`
- Create: `scripts/lib/reporter.js`

- [ ] **Step 1: Write the failing test**

Create `tests/reporter.test.js`:

```javascript
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { formatGithubSummary, formatSlackPayload } from '../scripts/lib/reporter.js';

const sampleResults = [
  { name: 'Login - Valid', status: 'pass', durationMs: 4200 },
  { name: 'Login - Invalid', status: 'fail', durationMs: 2100, error: 'Dashboard did not load' },
];

test('formatGithubSummary produces a markdown table with totals', () => {
  const md = formatGithubSummary({
    environment: 'UAT',
    results: sampleResults,
    runUrl: 'https://github.com/x/y/actions/runs/123',
  });

  assert.match(md, /Environment.*UAT/);
  assert.match(md, /\|.*Test.*\|.*Status.*\|.*Duration.*\|/);
  assert.match(md, /Login - Valid/);
  assert.match(md, /Login - Invalid/);
  assert.match(md, /Dashboard did not load/);
  assert.match(md, /Passed.*1/);
  assert.match(md, /Failed.*1/);
});

test('formatSlackPayload produces a JSON object with summary text', () => {
  const payload = formatSlackPayload({
    environment: 'INT',
    results: sampleResults,
    runUrl: 'https://github.com/x/y/actions/runs/123',
  });

  assert.equal(typeof payload.text, 'string');
  assert.match(payload.text, /INT/);
  assert.match(payload.text, /1 passed/);
  assert.match(payload.text, /1 failed/);
  assert.match(payload.text, /https:\/\/github\.com\/x\/y\/actions\/runs\/123/);
});

test('formatSlackPayload reports all-pass cleanly', () => {
  const payload = formatSlackPayload({
    environment: 'UAT',
    results: [{ name: 'A', status: 'pass', durationMs: 100 }],
    runUrl: 'https://example.com/run',
  });
  assert.match(payload.text, /1 passed/);
  assert.match(payload.text, /0 failed/);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: FAIL — `Cannot find module '../scripts/lib/reporter.js'`

- [ ] **Step 3: Write the implementation**

Create `scripts/lib/reporter.js`:

```javascript
function counts(results) {
  const passed = results.filter((r) => r.status === 'pass').length;
  const failed = results.filter((r) => r.status === 'fail').length;
  return { passed, failed };
}

export function formatGithubSummary({ environment, results, runUrl }) {
  const { passed, failed } = counts(results);
  const lines = [
    `# Test Run Results`,
    ``,
    `**Environment:** ${environment}`,
    `**Run:** ${runUrl}`,
    ``,
    `**Passed:** ${passed}  |  **Failed:** ${failed}`,
    ``,
    `| Test | Status | Duration | Notes |`,
    `| --- | --- | --- | --- |`,
  ];
  for (const r of results) {
    const icon = r.status === 'pass' ? '✅ pass' : '❌ fail';
    const dur = `${(r.durationMs / 1000).toFixed(1)}s`;
    const notes = r.error ? r.error.replace(/\|/g, '\\|').slice(0, 200) : '';
    lines.push(`| ${r.name} | ${icon} | ${dur} | ${notes} |`);
  }
  return lines.join('\n');
}

export function formatSlackPayload({ environment, results, runUrl }) {
  const { passed, failed } = counts(results);
  const headline = failed === 0 ? '✅ All tests passed' : '❌ Some tests failed';
  const text = `${headline}\nEnvironment: ${environment}\n${passed} passed, ${failed} failed\nRun: ${runUrl}`;
  return { text };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/vaseem/ai-powerd-automation && npm test`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/lib/reporter.js tests/reporter.test.js
git commit -m "feat: add reporter for GitHub summary and Slack payloads"
```

---

## Task 5: Playwright MCP Client Wrapper

**Files:**
- Create: `scripts/lib/playwrightMcpClient.js`

This wrapper starts the Playwright MCP server as a subprocess (stdio transport), connects via the MCP SDK, lists tools, and exposes a `callTool(name, args)` function. No unit test — verified via integration in Task 7.

- [ ] **Step 1: Write the implementation**

Create `scripts/lib/playwrightMcpClient.js`:

```javascript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

export async function startPlaywrightMcp() {
  const transport = new StdioClientTransport({
    command: 'npx',
    args: ['-y', '@playwright/mcp@latest', '--headless'],
  });

  const client = new Client(
    { name: 'ai-powerd-automation', version: '0.1.0' },
    { capabilities: {} }
  );

  await client.connect(transport);

  const { tools } = await client.listTools();

  const callTool = async (name, args) => {
    const result = await client.callTool({ name, arguments: args });
    return result;
  };

  const close = async () => {
    await client.close();
  };

  return { tools, callTool, close };
}
```

- [ ] **Step 2: Smoke test manually**

Run a quick smoke check that the MCP subprocess starts and lists tools:

```bash
cd /home/vaseem/ai-powerd-automation && node --input-type=module -e "
import { startPlaywrightMcp } from './scripts/lib/playwrightMcpClient.js';
const mcp = await startPlaywrightMcp();
console.log('Tool count:', mcp.tools.length);
console.log('First tool:', mcp.tools[0]?.name);
await mcp.close();
"
```

Expected: Prints a tool count > 5 and a tool name like `browser_navigate`. No errors.

- [ ] **Step 3: Commit**

```bash
git add scripts/lib/playwrightMcpClient.js
git commit -m "feat: add Playwright MCP stdio client wrapper"
```

---

## Task 6: Claude Agentic Loop

**Files:**
- Create: `scripts/lib/runTestWithClaude.js`

Runs one test case: builds a prompt with steps + credentials, sends to Claude with Playwright MCP tools as the tool schema, executes any tool calls Claude requests, loops until Claude returns a final pass/fail verdict.

- [ ] **Step 1: Write the implementation**

Create `scripts/lib/runTestWithClaude.js`:

```javascript
import Anthropic from '@anthropic-ai/sdk';

const MODEL = 'claude-sonnet-4-6';
const MAX_ITERATIONS = 30;

function mcpToolsToAnthropicSchema(mcpTools) {
  return mcpTools.map((t) => ({
    name: t.name,
    description: t.description ?? '',
    input_schema: t.inputSchema ?? { type: 'object', properties: {} },
  }));
}

function buildSystemPrompt({ baseUrl, credentials }) {
  return `You are a test automation agent. You execute test cases by calling Playwright MCP browser tools.

Base URL: ${baseUrl}
When a step references a URL path, prefix it with the base URL.

Available test users (use when a step says "from userdetails"):
${JSON.stringify(credentials.users, null, 2)}

Rules:
- Execute steps in order using browser tools.
- After completing all steps, evaluate the Expected Result against the page state.
- When you finish (pass or fail), respond with a single final message that starts with either "VERDICT: PASS" or "VERDICT: FAIL", followed by a one-line reason.
- If a tool call errors irrecoverably, return "VERDICT: FAIL" with the error.
- Do NOT include the verdict in any message that also contains tool calls.`;
}

function buildUserPrompt({ testCase }) {
  const stepsList = testCase.steps.map((s, i) => `${i + 1}. ${s}`).join('\n');
  return `Test: ${testCase.name}
Starting URL path: ${testCase.urlPath}

Steps:
${stepsList}

Expected Result:
${testCase.expectedResult}`;
}

export async function runTestWithClaude({ testCase, baseUrl, credentials, mcpClient, apiKey }) {
  const anthropic = new Anthropic({ apiKey });
  const tools = mcpToolsToAnthropicSchema(mcpClient.tools);
  const messages = [{ role: 'user', content: buildUserPrompt({ testCase }) }];
  const startedAt = Date.now();

  for (let i = 0; i < MAX_ITERATIONS; i++) {
    const response = await anthropic.messages.create({
      model: MODEL,
      max_tokens: 4096,
      system: buildSystemPrompt({ baseUrl, credentials }),
      tools,
      messages,
    });

    messages.push({ role: 'assistant', content: response.content });

    const toolUses = response.content.filter((b) => b.type === 'tool_use');
    if (toolUses.length === 0) {
      const textBlock = response.content.find((b) => b.type === 'text');
      const text = textBlock?.text ?? '';
      const status = /VERDICT:\s*PASS/i.test(text) ? 'pass' : 'fail';
      const reason = text.replace(/^[\s\S]*VERDICT:\s*(PASS|FAIL)\s*[:\-]?\s*/i, '').trim();
      return {
        status,
        durationMs: Date.now() - startedAt,
        reason,
      };
    }

    const toolResults = [];
    for (const tu of toolUses) {
      try {
        const result = await mcpClient.callTool(tu.name, tu.input);
        const contentBlocks = result.content ?? [{ type: 'text', text: 'ok' }];
        toolResults.push({
          type: 'tool_result',
          tool_use_id: tu.id,
          content: contentBlocks,
        });
      } catch (err) {
        toolResults.push({
          type: 'tool_result',
          tool_use_id: tu.id,
          is_error: true,
          content: [{ type: 'text', text: String(err.message ?? err) }],
        });
      }
    }
    messages.push({ role: 'user', content: toolResults });
  }

  return {
    status: 'fail',
    durationMs: Date.now() - startedAt,
    reason: `Exceeded max iterations (${MAX_ITERATIONS}) without verdict`,
  };
}
```

- [ ] **Step 2: Commit**

```bash
git add scripts/lib/runTestWithClaude.js
git commit -m "feat: add Claude agentic loop for test execution"
```

---

## Task 7: Main Runner Entry Point

**Files:**
- Create: `scripts/run-test.js`
- Create: `.env.example`

Discovers test files at a given path, runs each one, collects results, writes the GitHub summary, posts to Slack. Honors env vars for environment selection and secrets.

- [ ] **Step 1: Create `.env.example`**

```bash
# Used locally; in GitHub Actions these come from Secrets
ANTHROPIC_API_KEY=sk-ant-...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
TMP_UAT_URL=https://uat.tmp.example
TMP_INT_URL=https://int.tmp.example

# Set per run
ENVIRONMENT=UAT       # UAT or INT
TEST_PATH=test-cases/ # file or directory
```

- [ ] **Step 2: Write the implementation**

Create `scripts/run-test.js`:

```javascript
import { readFileSync, statSync, readdirSync, writeFileSync, existsSync, appendFileSync } from 'node:fs';
import { join, resolve } from 'node:path';
import { parseTestCase } from './lib/parseTestCase.js';
import { loadCredentials } from './lib/loadCredentials.js';
import { startPlaywrightMcp } from './lib/playwrightMcpClient.js';
import { runTestWithClaude } from './lib/runTestWithClaude.js';
import { formatGithubSummary, formatSlackPayload } from './lib/reporter.js';

function discoverTestFiles(target) {
  const abs = resolve(target);
  const stat = statSync(abs);
  if (stat.isFile()) return [abs];
  const out = [];
  const walk = (dir) => {
    for (const entry of readdirSync(dir, { withFileTypes: true })) {
      const p = join(dir, entry.name);
      if (entry.isDirectory()) walk(p);
      else if (entry.isFile() && p.endsWith('.md')) out.push(p);
    }
  };
  walk(abs);
  return out.sort();
}

function envFor(name) {
  const value = process.env[name];
  if (!value) throw new Error(`Required environment variable not set: ${name}`);
  return value;
}

async function postSlack(webhookUrl, payload) {
  const res = await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
  if (!res.ok) {
    console.error(`Slack webhook failed: ${res.status} ${await res.text()}`);
  }
}

async function main() {
  const environment = (process.env.ENVIRONMENT ?? 'UAT').toUpperCase();
  const testPath = process.env.TEST_PATH ?? 'test-cases/';
  const apiKey = envFor('ANTHROPIC_API_KEY');
  const baseUrl = environment === 'INT' ? envFor('TMP_INT_URL') : envFor('TMP_UAT_URL');
  const slackUrl = process.env.SLACK_WEBHOOK_URL;
  const runUrl =
    process.env.GITHUB_SERVER_URL && process.env.GITHUB_REPOSITORY && process.env.GITHUB_RUN_ID
      ? `${process.env.GITHUB_SERVER_URL}/${process.env.GITHUB_REPOSITORY}/actions/runs/${process.env.GITHUB_RUN_ID}`
      : 'local';

  const credentials = loadCredentials('config/userdetails.json');
  const files = discoverTestFiles(testPath);
  if (files.length === 0) {
    console.error(`No test files found at ${testPath}`);
    process.exit(1);
  }

  console.log(`Running ${files.length} test(s) on ${environment} (${baseUrl})`);

  const mcp = await startPlaywrightMcp();
  const results = [];

  try {
    for (const file of files) {
      const md = readFileSync(file, 'utf8');
      let testCase;
      try {
        testCase = parseTestCase(md);
      } catch (err) {
        results.push({
          name: file,
          status: 'fail',
          durationMs: 0,
          error: `Parse error: ${err.message}`,
        });
        continue;
      }

      if (testCase.environment !== 'both' && testCase.environment.toUpperCase() !== environment) {
        console.log(`Skipping ${testCase.name} (not for ${environment})`);
        continue;
      }

      console.log(`▶ ${testCase.name}`);
      const result = await runTestWithClaude({
        testCase,
        baseUrl,
        credentials,
        mcpClient: mcp,
        apiKey,
      });
      console.log(`  → ${result.status} (${(result.durationMs / 1000).toFixed(1)}s) ${result.reason ?? ''}`);
      results.push({
        name: testCase.name,
        status: result.status,
        durationMs: result.durationMs,
        error: result.status === 'fail' ? result.reason : undefined,
      });
    }
  } finally {
    await mcp.close();
  }

  const summary = formatGithubSummary({ environment, results, runUrl });
  console.log('\n' + summary);

  if (process.env.GITHUB_STEP_SUMMARY) {
    appendFileSync(process.env.GITHUB_STEP_SUMMARY, summary + '\n');
  }
  writeFileSync('reports/summary.md', summary);

  if (slackUrl) {
    await postSlack(slackUrl, formatSlackPayload({ environment, results, runUrl }));
  }

  const anyFailed = results.some((r) => r.status === 'fail');
  process.exit(anyFailed ? 1 : 0);
}

main().catch((err) => {
  console.error('Runner crashed:', err);
  process.exit(2);
});
```

- [ ] **Step 3: Verify it loads without runtime error (no API key needed for module load)**

Run: `cd /home/vaseem/ai-powerd-automation && node --check scripts/run-test.js`
Expected: No output (syntax OK).

- [ ] **Step 4: Commit**

```bash
git add scripts/run-test.js .env.example
git commit -m "feat: add main test runner entry point"
```

---

## Task 8: Sample Test Case + Sanitize `.mcp.json`

**Files:**
- Create: `test-cases/login/login-basic.md`
- Modify: `.mcp.json`

- [ ] **Step 1: Create the sample test case**

Create `test-cases/login/login-basic.md`:

```markdown
---
name: Login - Valid Credentials
environment: both
url_path: /login
---

## Steps

1. Navigate to the login page using the starting URL path
2. Enter the standard user email from userdetails into the email field
3. Enter the standard user password from userdetails into the password field
4. Click the "Sign In" button
5. Wait for the page to load
6. Take a screenshot

## Expected Result
The user is logged in and the dashboard page is visible.
```

- [ ] **Step 2: Sanitize `.mcp.json`**

The current `.mcp.json` contains hardcoded Airtable API keys that must NOT be committed to a remote repo. Replace the file contents with this minimal version (Playwright MCP only — the only one the runner uses):

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

- [ ] **Step 3: Commit**

```bash
git add test-cases/login/login-basic.md .mcp.json
git commit -m "feat: add sample login test case and sanitize mcp config"
```

---

## Task 9: GitHub Workflow — Auto Trigger

**Files:**
- Create: `.github/workflows/run-tests.yml`

- [ ] **Step 1: Write the workflow**

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
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          TMP_UAT_URL: ${{ secrets.TMP_UAT_URL }}
          TMP_INT_URL: ${{ secrets.TMP_INT_URL }}
          ENVIRONMENT: UAT
          TEST_PATH: test-cases/
        run: npm run run-test

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: reports/
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/run-tests.yml
git commit -m "ci: add auto-trigger workflow for push and PR"
```

---

## Task 10: GitHub Workflow — Manual Trigger

**Files:**
- Create: `.github/workflows/manual-run.yml`

- [ ] **Step 1: Write the workflow**

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
        description: 'Test file or directory (e.g. test-cases/login/login-basic.md)'
        required: true
        default: 'test-cases/'
        type: string

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          TMP_UAT_URL: ${{ secrets.TMP_UAT_URL }}
          TMP_INT_URL: ${{ secrets.TMP_INT_URL }}
          ENVIRONMENT: ${{ inputs.environment }}
          TEST_PATH: ${{ inputs.test_path }}
        run: npm run run-test

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: reports/
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/manual-run.yml
git commit -m "ci: add manual workflow with environment selector"
```

---

## Task 11: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace `README.md` contents**

```markdown
# ai-powerd-automation

AI-powered test automation framework. Claude (via the Anthropic API) reads plain-English test cases from `.md` files and drives a headless Chromium browser via Playwright MCP. Runs on GitHub Actions.

## How it works

1. You write a test case in plain English in a `.md` file under `test-cases/`.
2. GitHub Actions runs the test (auto on push/PR, or manual via the Actions tab).
3. The runner sends your steps to Claude, which calls Playwright MCP browser tools to execute them.
4. Results land in the GitHub Actions summary and a Slack channel.

## Test case format

```markdown
---
name: Login - Valid Credentials
environment: both       # UAT, INT, or both
url_path: /login
---

## Steps

1. Navigate to the login page
2. Enter the standard user email from userdetails
3. Enter the standard user password from userdetails
4. Click the "Sign In" button

## Expected Result
The user is logged in and the dashboard page is visible.
```

## Project layout

- `test-cases/` — your test cases (`.md`)
- `config/userdetails.json` — test user credentials
- `scripts/` — runner code
- `.github/workflows/` — auto and manual CI workflows

## Required GitHub secrets

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude API |
| `SLACK_WEBHOOK_URL` | Slack notifications |
| `TMP_UAT_URL` | UAT base URL |
| `TMP_INT_URL` | INT base URL |

## Running locally

```bash
npm install
npx playwright install chromium
cp .env.example .env   # fill in values
ENVIRONMENT=UAT TEST_PATH=test-cases/login/login-basic.md npm run run-test
```

## Running unit tests

```bash
npm test
```
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README with usage instructions"
```

---

## Task 12: End-to-End Smoke Verification

This is a manual verification gate — confirms the runner can complete one real test against a live URL before relying on CI.

- [ ] **Step 1: Confirm `.env` is filled in**

Check `/home/vaseem/ai-powerd-automation/.env` contains real values for `ANTHROPIC_API_KEY`, `TMP_UAT_URL`, and that `config/userdetails.json` has real credentials. (Do NOT commit `.env`.)

- [ ] **Step 2: Run the sample test locally**

```bash
cd /home/vaseem/ai-powerd-automation
set -a; source .env; set +a
ENVIRONMENT=UAT TEST_PATH=test-cases/login/login-basic.md npm run run-test
```

Expected: Console shows the test name, then either `pass` or `fail` with a reason. `reports/summary.md` is written. Exit code is 0 on pass, 1 on fail.

- [ ] **Step 3: If the test passed, you're done**

Push the branch and watch the GitHub Actions run on `main`. Configure secrets in GitHub repo settings before pushing.

- [ ] **Step 4: Final commit (if any tweaks were needed)**

If you adjusted anything during smoke testing:

```bash
git add -A
git commit -m "chore: smoke-test fixes"
```

---

## Self-Review Notes

**Spec coverage:** Every section of the spec maps to a task — repo structure (1, 7, 8), data flow (5, 6, 7), test format (8 + parser in 2), credentials (3), workflows (9, 10), secrets (handled in workflow env blocks).

**Type consistency:** `parseTestCase` returns `{name, environment, urlPath, steps, expectedResult}` — `runTestWithClaude` and `run-test.js` consume those exact fields. `mcpClient` shape is `{tools, callTool, close}` defined in Task 5 and used in Tasks 6 and 7.

**Security note:** Task 8 explicitly removes the hardcoded Airtable keys from `.mcp.json` before they could be pushed to GitHub. `userdetails.json` is committed per user preference.
