---
description: "Run Playwright E2E tests derived from OpenSpec scenarios using parallel workers"
argument-hint: "[spec-path] [--headed] [--browsers chromium,firefox,webkit]"
allowed-tools: ["Task", "TaskOutput", "Bash", "Read", "Write", "Glob", "Grep", "AskUserQuestion"]
---

# Run E2E Tests Command

Execute browser-based E2E tests derived from OpenSpec Given/When/Then scenarios using parallel Playwright workers.

## Overview

This command:
1. Verifies Playwright is installed (or installs it)
2. Selects an OpenSpec change to test
3. Derives E2E tests from Given/When/Then scenarios
4. Handles authentication with storage state persistence
5. Runs tests in parallel using swarm workers
6. Reports results and creates beads for failures

## Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `[spec-path]` | (interactive) | Optional path to OpenSpec directory (active or archived) |
| `--headed` | false | Run browsers in visible mode |
| `--browsers` | chromium | Comma-separated: chromium,firefox,webkit |

### Examples

```bash
# Interactive selection from active OpenSpecs
/run-e2e

# Explicit path to active spec
/run-e2e openspec/changes/change-001

# Explicit path to archived spec
/run-e2e openspec/changes/archive/change-001

# With options
/run-e2e openspec/changes/archive/change-001 --headed --browsers chromium,firefox
```

## Instructions

### Step 1: Verify Playwright Installation

Check if Playwright is installed:

```bash
npx playwright --version
```

If not installed, ask user:

```
Playwright is not installed. Would you like to install it?
- Yes, install Playwright (Recommended)
- No, cancel
```

If yes:
```bash
npm install -D @playwright/test && npx playwright install
```

### Step 2: Parse Arguments

Parse command arguments:
- `--headed`: Set headed mode flag
- `--browsers`: Parse comma-separated browser list (default: chromium)

Valid browsers: chromium, firefox, webkit

### Step 3: Check Strategy File

Look for `playwright.config.md` in project root:

```bash
test -f playwright.config.md && echo "EXISTS" || echo "NOT_FOUND"
```

If not found, create it and ask for BASE_URL:

```
No E2E strategy file found. I need to create playwright.config.md.

What is the base URL of your application?
(e.g., http://localhost:3000 or https://staging.myapp.com)
```

Create `playwright.config.md`:

```markdown
# Playwright E2E Configuration

## Application URLs

| Setting | Value |
|---------|-------|
| BASE_URL | <user-provided-url> |
| LOGIN_URL | /login |

## Authentication Selectors

| Element | Selector |
|---------|----------|
| Email Input | input[name="email"] |
| Password Input | input[name="password"] |
| Submit Button | button[type="submit"] |

## Timeouts

| Setting | Value |
|---------|-------|
| Page Load | 30000 |
| Element Visible | 10000 |
| Navigation | 15000 |

## Notes

Edit this file to customize selectors and timeouts for your application.
```

### Step 4: Verify BASE_URL is Reachable

Parse BASE_URL from playwright.config.md and verify it's reachable:

```bash
curl -s -o /dev/null -w "%{http_code}" <BASE_URL> --max-time 10
```

**If not reachable (non-2xx/3xx response or timeout), BLOCK with clear error:**

```
===============================================================
ERROR: Application not reachable
===============================================================

BASE_URL: <url>
Status: <status or "Connection timed out">

Please ensure your application is running before running E2E tests.

To update the URL, edit: playwright.config.md
===============================================================
```

**Do NOT proceed until URL is reachable.**

### Step 4.5: Detect Test Directory from Config

If `playwright.config.ts` exists, parse it to detect the test directory structure:

```bash
test -f playwright.config.ts && echo "CONFIG_EXISTS" || echo "NO_CONFIG"
```

**If config exists**, read it and extract:

1. **Authenticated project's testDir**: Look for a project with `name` containing "auth":
   ```typescript
   projects: [
     {
       name: "chromium-auth",  // or any name containing "auth"
       testDir: "./tests/authenticated",  // ← Extract this
       use: { storageState: "..." }
     }
   ]
   ```

2. **Storage state path**: From the authenticated project's `use.storageState`

3. **Fallback to root testDir**: If no authenticated project found, use root `testDir`

**Detection priority:**
1. Authenticated project's `testDir` (e.g., `./tests/authenticated`)
2. Root `testDir` setting
3. Default: `./tests/e2e`

**Store detected values:**
- `TEST_DIR` = detected test directory (e.g., `apps/web/tests/authenticated`)
- `STORAGE_STATE_PATH` = detected storage state path (e.g., `playwright/.auth/user.json`)
- `CONFIG_EXISTS` = true/false

**If no config exists**, use defaults:
- `TEST_DIR` = `./tests/e2e`
- `STORAGE_STATE_PATH` = `.auth/state.json`
- `CONFIG_EXISTS` = false

### Step 5: Select OpenSpec Change

**If spec-path argument was provided:**

Validate the provided path:

```bash
# Check directory exists
test -d "<spec-path>" && echo "EXISTS" || echo "NOT_FOUND"

# Check specs subdirectory exists
test -d "<spec-path>/specs" && echo "HAS_SPECS" || echo "NO_SPECS"

# Check for spec files
find "<spec-path>/specs" -name "*.md" -type f 2>/dev/null | head -1
```

If validation fails:
- Directory doesn't exist → Show error: "OpenSpec path not found: `<spec-path>`"
- No specs/ subdirectory → Show error: "No specs/ directory in `<spec-path>`"
- No .md files in specs/ → Show error: "No spec files found in `<spec-path>/specs`"

If valid, use the provided path and skip interactive selection.

**If no spec-path argument (interactive mode):**

List available OpenSpec changes:

```bash
ls -d openspec/changes/*/ 2>/dev/null | grep -v '/archive/'
```

Present interactive selection:

```
Select an OpenSpec change to test:
- change-001: <title from proposal.md>
- change-002: <title from proposal.md>
- ...
```

Read the selected change's proposal.md to get the title.

### Step 6: Check Existing Tests

Check for existing tests in the detected `TEST_DIR`:

```bash
ls ${TEST_DIR}/*.spec.ts 2>/dev/null || echo "NONE"
```

If tests exist for this change, ask:

```
Existing E2E tests found:
- <list of files>

What would you like to do?
- Regenerate all tests (Recommended)
- Keep existing tests
- Merge (keep existing, add new scenarios)
```

### Step 7: Parse OpenSpec Scenarios

Read all spec files from the selected change (using the path from Step 5):

```bash
find "<spec-path>/specs" -name "*.md" -type f
```

For each spec file, extract Given/When/Then scenarios:

```
Requirement: <Name>
  Scenario: <Name>
    GIVEN <precondition>
    WHEN <action>
    THEN <outcome>
```

Build scenario list:
```json
[
  {
    "requirement": "<name>",
    "scenario": "<name>",
    "given": "<context>",
    "when": "<action>",
    "then": "<expected>",
    "spec_file": "<path>"
  }
]
```

### Step 8: Generate Test Files

Create the test directory if needed (using `TEST_DIR` from Step 4.5):

```bash
mkdir -p ${TEST_DIR}
```

#### Step 8.1: Analyze Existing Tests for Patterns

**Before generating tests**, read 2-3 existing test files from `${TEST_DIR}` to learn project-specific patterns:

```bash
ls ${TEST_DIR}/*.spec.ts 2>/dev/null | head -3
```

For each existing test file, extract and adopt:

| Pattern to Learn | Example |
|------------------|---------|
| **Import style** | `import { test, expect } from "@playwright/test"` (quotes, semicolons) |
| **Selector conventions** | `data-slot="..."`, `data-testid="..."`, or semantic selectors |
| **Wait strategies** | `waitForLoadState("networkidle")`, `waitForTimeout(2000)` |
| **Timeout values** | `toBeVisible({ timeout: 15000 })` |
| **URL patterns** | `/en/articles` (locale prefix), `/api/v1/...` |
| **Test organization** | Multiple `test.describe` blocks, nesting patterns |
| **Assertion style** | `expect(locator).toBeVisible()` vs `expect(await locator.isVisible()).toBe(true)` |
| **Common helpers** | Shared setup, custom fixtures |

**Critical:** Match the existing code style exactly:
- Same quote style (single vs double)
- Same semicolon usage
- Same indentation (tabs vs spaces)
- Same `test.describe` organization patterns

#### Step 8.2: Generate Tests Using Learned Patterns

Generate Playwright test files following the patterns learned from existing tests.

**Default Translation Rules** (override with patterns from Step 8.1):

| Pattern | Playwright Code |
|---------|-----------------|
| GIVEN user is on /path | `await page.goto('/path')` |
| GIVEN user is logged in | `// Uses storage state` |
| WHEN user clicks "X" | `await page.click('text=X')` |
| WHEN user clicks button "X" | `await page.click('button:has-text("X")')` |
| WHEN user fills "X" with "Y" | `await page.fill('[placeholder="X"]', 'Y')` |
| WHEN user types "X" in field "Y" | `await page.fill('input[name="Y"]', 'X')` |
| WHEN user submits form | `await page.click('button[type="submit"]')` |
| WHEN user navigates to /path | `await page.goto('/path')` |
| THEN user sees "X" | `await expect(page.locator('text=X')).toBeVisible()` |
| THEN page contains "X" | `await expect(page.locator('text=X')).toBeVisible()` |
| THEN user is redirected to /path | `await expect(page).toHaveURL(/.*\/path/)` |
| THEN button "X" is disabled | `await expect(page.locator('button:has-text("X")')).toBeDisabled()` |
| THEN input "X" has value "Y" | `await expect(page.locator('input[name="X"]')).toHaveValue('Y')` |

**Test file template** (adapt to match existing test style):

```typescript
import { test, expect } from '@playwright/test';

test.describe('<Requirement Name>', () => {
  test('<Scenario Name>', async ({ page }) => {
    // GIVEN <given>
    <given_code>

    // WHEN <when>
    <when_code>

    // THEN <then>
    <then_code>
  });

  // ... more scenarios
});
```

Write files to `${TEST_DIR}/<requirement-slug>.spec.ts`

### Step 9: Check Authentication State

Check if valid storage state exists (using `STORAGE_STATE_PATH` from Step 4.5):

```bash
if [ -f "${STORAGE_STATE_PATH}" ]; then
  # Check if less than 24 hours old
  find "${STORAGE_STATE_PATH}" -mtime -1 -type f | grep -q . && echo "VALID" || echo "EXPIRED"
else
  echo "NOT_FOUND"
fi
```

If VALID, skip to Step 11.

### Step 10: Authenticate

If no valid storage state, ask for credentials:

```
Authentication required for E2E tests.

Please provide test credentials:
- Email/Username
- Password
```

**If `CONFIG_EXISTS` is true** (from Step 4.5), the project already has authentication configured. Check if the detected storage state exists:

```bash
test -f "${STORAGE_STATE_PATH}" && echo "AUTH_EXISTS" || echo "NEEDS_AUTH"
```

If `NEEDS_AUTH`, inform the user:
```
Storage state not found at: ${STORAGE_STATE_PATH}
Please run your project's authentication setup before running E2E tests.
```

**If `CONFIG_EXISTS` is false**, create auth setup script at the detected path's parent directory:

```typescript
// File: <parent of STORAGE_STATE_PATH>/setup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto('<BASE_URL><LOGIN_URL>');
  await page.fill('<email_selector>', '<email>');
  await page.fill('<password_selector>', '<password>');
  await page.click('<submit_selector>');

  // Wait for authentication to complete
  await page.waitForURL('**/*', { timeout: 30000 });

  // Save storage state
  await page.context().storageState({ path: '${STORAGE_STATE_PATH}' });
  await browser.close();
}

export default globalSetup;
```

**Important:** Do NOT run the setup script directly as a test. It must be referenced in `playwright.config.ts` (see Step 11).

Create the auth directory and add to `.gitignore`:

```bash
mkdir -p $(dirname "${STORAGE_STATE_PATH}")
echo "$(dirname "${STORAGE_STATE_PATH}")/" >> .gitignore
```

The auth setup will run automatically via `globalSetup` when you run `npx playwright test`.

**To manually trigger auth setup** (if storage state is missing/expired):

```bash
# Delete existing state to force re-auth
rm -f "${STORAGE_STATE_PATH}"

# Run any test - globalSetup will execute first
npx playwright test --grep ".*" --max-failures=1
```

Verify success:
```bash
test -f "${STORAGE_STATE_PATH}" && echo "AUTH_SUCCESS" || echo "AUTH_FAILED"
```

If AUTH_FAILED, show error and ask user to check credentials/selectors in playwright.config.md.

### Step 11: Create Playwright Config (if needed)

**If `CONFIG_EXISTS` is true** (from Step 4.5), skip this step entirely. The project already has a Playwright configuration.

**If `CONFIG_EXISTS` is false**, create `playwright.config.ts`:

```typescript
import { defineConfig, devices } from '@playwright/test';
import * as fs from 'fs';

// Conditionally use storage state only if it exists
const storageStatePath = '${STORAGE_STATE_PATH}';
const useStorageState = fs.existsSync(storageStatePath) ? { storageState: storageStatePath } : {};

export default defineConfig({
  testDir: '${TEST_DIR}',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results.json' }]
  ],

  // Global setup for authentication
  globalSetup: require.resolve('./${STORAGE_STATE_PATH}/../setup.ts'),

  use: {
    baseURL: '<BASE_URL>',
    trace: 'on-first-retry',
    ...useStorageState,
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

**Note:** The config uses conditional `storageState` to avoid errors when `.auth/state.json` doesn't exist yet. The `globalSetup` ensures auth runs before tests.

### Step 12: Launch Swarm Workers

Get list of test files:

```bash
ls tests/e2e/*.spec.ts
```

For each browser in --browsers list, for each test file, spawn a worker:

**Worker spawn pattern (max 5 concurrent):**

```
WHILE (pending_files > 0 OR active_workers > 0):
    1. If active_workers < 5 AND pending_files > 0:
       - Pop next file from pending
       - Spawn worker with Task tool:
         - subagent_type: "architect:e2e-worker"
         - run_in_background: true
         - prompt: Include test_file, browser, headed_mode
       - Add task_id to active_workers

    2. Wait for any worker to complete:
       - TaskOutput(task_id, block=true, timeout=120000)

    3. Parse result, add to passed/failed lists

    4. Remove completed worker from active list

    5. Report progress:
       "Completed: X/Y | Passed: P | Failed: F"
END WHILE
```

### Step 13: Consolidate Results

After all workers complete, output results:

**If all tests passed:**

```
===============================================================
E2E TESTS PASSED
===============================================================

Tests Run: <total>
Passed: <passed>
Failed: 0
Duration: <total_duration>

Browsers tested: <browser_list>

HTML Report: playwright-report/index.html
  Open with: npx playwright show-report

===============================================================
```

**If any tests failed:**

```
===============================================================
E2E TESTS FAILED
===============================================================

Tests Run: <total>
Passed: <passed>
Failed: <failed>

FAILURES:
---------

<test_file>
  ✗ <test_name>
    Expected: <expected>
    Received: <actual>
    Screenshot: <path>

... (more failures)

HTML Report: playwright-report/index.html
  Open with: npx playwright show-report

===============================================================
```

Ask user:

```
What would you like to do?
- Create bead for failures (Recommended)
- View HTML report
- Done
```

If "Create bead":

```bash
bd create "Fix E2E test failures in <change-id>" -t task -p 1 -l e2e-failures
```

Include failure details in bead description.

## Error Handling

| Error | Action |
|-------|--------|
| Playwright not installed | Offer to install |
| URL not reachable | BLOCK with clear message |
| Spec-path not found | Show: "OpenSpec path not found: `<path>`" |
| Spec-path has no specs/ | Show: "No specs/ directory in `<path>`" |
| No OpenSpec changes (interactive) | Show: "No OpenSpec changes found. Run /architect:create-openspec first" |
| No specs/*.md files | Show: "No scenarios found in specs/ directory" |
| Auth fails | Show error, suggest checking credentials/selectors |
| Worker timeout | Mark as failed, continue with others |
| All workers fail | Show summary, offer to create bead |

## File Structure

After running this command (paths vary based on project configuration):

**New project (no existing config):**
```
project/
├── .gitignore                # Should contain auth directory entry
├── playwright.config.md      # Strategy file (user-maintained)
├── playwright.config.ts      # Playwright config (generated)
├── .auth/                    # IMPORTANT: Add to .gitignore!
│   ├── setup.ts              # Auth setup script (globalSetup)
│   └── state.json            # Storage state (reused <24h)
├── tests/
│   └── e2e/
│       ├── <capability>.spec.ts
│       └── ...
├── test-results.json         # JSON reporter output
└── playwright-report/
    └── index.html            # HTML report
```

**Existing project (e.g., monorepo with auth config):**
```
project/
├── playwright.config.ts      # Existing config (not modified)
├── playwright/
│   └── .auth/
│       └── user.json         # Existing storage state path
├── apps/
│   └── web/
│       └── tests/
│           └── authenticated/    # Detected TEST_DIR
│               ├── <capability>.spec.ts
│               └── ...
└── test-results.json
```

**Security Note:** The auth directory (wherever located) contains credentials and tokens. Always ensure it's in `.gitignore`.

## Example Usage

```bash
# Run with defaults (headless chromium, interactive selection)
/architect:run-e2e

# Run with visible browser
/architect:run-e2e --headed

# Run on multiple browsers
/architect:run-e2e --browsers chromium,firefox

# Run headed on all browsers
/architect:run-e2e --headed --browsers chromium,firefox,webkit

# Run on specific OpenSpec (skip interactive selection)
/architect:run-e2e openspec/changes/change-001

# Run on archived OpenSpec
/architect:run-e2e openspec/changes/archive/change-001

# Run archived spec with options
/architect:run-e2e openspec/changes/archive/change-001 --headed --browsers chromium,firefox
```
