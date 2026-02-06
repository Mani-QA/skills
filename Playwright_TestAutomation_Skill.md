---
name: playwright-testing
description: >
  Production-grade Playwright test automation with TypeScript (v1.50+). Use this skill whenever
  generating, reviewing, or refactoring Playwright test code, Page Object Models, custom fixtures,
  test configurations, or CI/CD pipelines for Playwright. Triggers include: any mention of
  "Playwright", "e2e test", "end-to-end test", "browser test", "test automation", POM/Page Object,
  test fixtures, sharding, blob reports, flaky test remediation, or accessibility-first testing.
  Also use when writing GitHub Actions workflows for Playwright, configuring playwright.config.ts,
  or enforcing strict TypeScript in automation projects.
---

# Playwright Engineering Standards (v1.50+)

## 1. Architecture & Philosophy

### 1.1 Engineering-Grade Automation

Playwright v1.50+ represents a paradigm shift from "scripts" to "systems." AI assistants must
architect automation that leverages dependency injection, network interception, accessibility-first
locators, and parallel orchestration. The goal: code that is **resilient**, **modular**,
**deterministic**, **strictly typed**, and **scalable**.

### 1.2 Chrome for Testing (v1.57+)

Playwright switched from generic Chromium to **Chrome for Testing** builds — version-locked,
automation-optimized binaries mirroring the stable release channel. This eliminates
bleeding-edge browser quirks as a flakiness vector.

**Strategic implication:** When debugging rendering issues, prioritize application code analysis
over browser inconsistency theories. The engine now mirrors production more closely than ever.

### 1.3 Write "Healable" Code — The AI-Native Lifecycle

Playwright Agents (v1.55+) introduced Planner, Generator, and Healer — AI agents that create
and self-repair tests. The Healer analyzes DOM diffs and patches source code automatically.

**Directive:** Write code the Healer can re-contextualize. Semantic, intent-based locators
(`getByRole`) are transparent to AI agents. Brittle XPath/CSS selectors are opaque and break
the AI-native lifecycle. Every locator choice must ask: "Can an AI agent understand and fix this
if the DOM changes?"

### 1.4 The Accessibility-First Mandate

The removal of `page.accessibility()` in v1.57 enforces accessibility as the **primary interaction
mechanism**, not a separate concern. The Accessibility Tree (via `getByRole`) is the canonical
map of the application.

If an element cannot be targeted by role and accessible name, the AI must **suggest
accessibility fixes** (add `aria-label`, explicit `<label>` tags, semantic HTML) rather than creating
fragile CSS workarounds.

---

## 2. Tech Stack

- **Framework**: `@playwright/test` (latest stable, v1.50+)
- **Language**: TypeScript (`strict: true`)
- **CI/CD**: GitHub Actions
- **Browsers**: Chrome for Testing (default), Firefox, WebKit
- **Node.js**: 20 LTS or later (Node 18 deprecated)

---

## 3. The Semantic Locator Protocol

### 3.1 Priority Ladder (Strict Order of Operations)

Fallback to a lower tier requires explicit justification in code comments.

| Priority | Locator | Stability | AI Directive |
|----------|---------|-----------|-------------|
| 1 | `getByRole()` | High | **MANDATORY** if applicable. Represents the user's mental model. |
| 2 | `getByLabel()` | High | Use for form inputs. Requires proper `<label>` or `aria-label`. |
| 3 | `getByPlaceholder()` | Medium | Acceptable when labels are absent but placeholders are stable. |
| 4 | `getByText()` | Medium | Non-interactive content. Avoid for buttons (use Role). |
| 5 | `getByTestId()` | High | Only when semantic locators are impossible (e.g., virtualized lists). |
| 6 | `locator(css)` | Low | **AVOID.** Permitted only for styling verification or legacy patching. |
| 7 | `locator(xpath)` | Critical | **FORBIDDEN.** Represents a failure of the DOM to be testable. |

**Why `getByRole` is supreme:** `getByRole('button', { name: 'Submit' })` survives if the element
changes from `<button>` to `<div role="button">`, or if CSS classes are rewritten. It fails only if
the *intent* changes — which is a valid test failure.

```typescript
// ✅ Best — role-based (survives refactors)
await page.getByRole('button', { name: 'Submit' }).click();
await page.getByRole('textbox', { name: 'Email' }).fill('user@test.com');
await page.getByRole('checkbox', { name: 'I agree' }).check();

// ✅ Good — label-based for forms
await page.getByLabel('Password').fill('secret');

// ✅ Acceptable — test ID as fallback (with comment justifying why)
await page.getByTestId('virtualized-row-3').click(); // No semantic role on virtual scroll item

// ❌ AVOID
await page.locator('#btn-submit').click();
await page.locator('//div[@class="form"]/button[1]').click();
```

### 3.2 Advanced Filtering and Chaining

**Rule:** Never use `.first()`, `.last()`, or `.nth(n)` unless element order IS the explicit assertion.
Use `.filter()` to narrow by content or descendant state.

```typescript
// ❌ Structural reliance — breaks if item moves
await page.locator('.product-item').nth(3).getByRole('button').click();

// ✅ Semantic filtering — survives reordering
await page.getByRole('listitem')
  .filter({ hasText: 'Gaming Laptop' })
  .filter({ has: page.getByRole('img', { name: 'In Stock' }) })
  .getByRole('button', { name: 'Add to Cart' })
  .click();

// ✅ Combine locators with .and()
const subscribeBtn = page.getByRole('button').and(page.getByTitle('Subscribe'));

// ✅ Filter rows by cell content
await page.getByRole('row')
  .filter({ has: page.getByRole('cell', { name: 'Active' }) })
  .getByRole('button', { name: 'Edit' })
  .click();
```

### 3.3 Selecting vs. Asserting Implementation Details

Distinguish between *finding* an element and *asserting its state*. Never use CSS classes to
locate interactive elements, but you MAY assert class-based state.

```typescript
// ❌ WRONG — selecting by class
await page.locator('.is-active').click();

// ✅ CORRECT — semantic selection, class assertion
const tab = page.getByRole('tab', { name: 'Settings' });
await tab.click();
await expect(tab).toContainClass('active');   // v1.52+ — asserting visual state

// ✅ BETTER if ARIA attributes exist
await expect(page.getByRole('tab', { name: 'Settings', selected: true })).toBeVisible();
```

---

## 4. The Auto-Waiting Engine

### 4.1 Actionability Checks

Before `.click()`, Playwright automatically verifies the element is:

1. **Attached** to the DOM
2. **Visible** (computed style is not hidden)
3. **Stable** (not animating or moving)
4. **Receives events** (not obscured by a modal/overlay)
5. **Enabled** (no `disabled` attribute)

This is why `waitForTimeout()` is never needed and why `page.evaluate(() => el.click())` is
forbidden — it bypasses all five checks.

### 4.2 Never Use Hard Waits

```typescript
// ❌ FORBIDDEN — hard waiting
await page.waitForTimeout(3000);
await new Promise(resolve => setTimeout(resolve, 1000));

// ✅ Auto-retrying web-first assertion (retries for expect timeout, default 5s)
await expect(page.getByRole('status')).toHaveText('Complete');

// ✅ Complex conditions — retries entire block
await expect(async () => {
  const response = await page.request.get('/api/status');
  expect(response.status()).toBe(200);
}).toPass({ timeout: 10_000 });
```

### 4.3 Never Assert on Snapshots

```typescript
// ❌ Snapshot in time — no auto-wait
const isVisible = await page.locator('#status').isVisible();
expect(isVisible).toBe(true);

// ✅ Web-first — polls until condition met or timeout
await expect(page.getByText('Order confirmed')).toBeVisible();
```

### 4.4 Always Assert the Effect of Actions ("No Blind Clicks")

Every user action must be followed by an assertion verifying its effect.

```typescript
// ❌ Blind click — did it work? did the page change?
await page.getByRole('button', { name: 'Save' }).click();

// ✅ Assert the effect
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByText('Saved successfully')).toBeVisible();
```

### 4.5 Soft Assertions for Bulk Validation

When validating multiple fields, collect all failures instead of stopping at the first.

```typescript
test('form displays all validation errors', async ({ page }) => {
  await page.getByRole('button', { name: 'Submit' }).click();

  await expect.soft(page.getByText('Name is required')).toBeVisible();
  await expect.soft(page.getByText('Email is required')).toBeVisible();
  await expect.soft(page.getByText('Password is required')).toBeVisible();
  // Test reports ALL failures, not just the first
});
```

### 4.6 Custom Expect Messages

Provide context in complex assertions for clear failure output.

```typescript
await expect(
  page.getByRole('heading'),
  'Dashboard heading should appear after login'
).toHaveText('Welcome back');
```

---

## 5. Dependency Injection: The Fixture System

### 5.1 The "No-New" Rule

The `new` keyword must NEVER appear in test bodies. Dependencies are declared in the test
signature and injected via `test.extend()`.

```typescript
// ❌ ANTI-PATTERN — manual instantiation in test
test('login', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.navigate();
});

// ✅ v1.50+ STANDARD — fixture injection
test('login', async ({ loginPage }) => {
  await loginPage.navigate();
});
```

### 5.2 Page Object Class Definition

```typescript
// pages/login.page.ts
import type { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(private readonly page: Page) {
    this.usernameInput = page.getByLabel('Username');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### 5.3 Custom Fixture Infrastructure

```typescript
// fixtures/index.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';

type AppFixtures = {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

export const test = base.extend<AppFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect };
```

### 5.4 Auto-Authentication Fixture

If a test requires "logged in" state, encapsulate it in a fixture — not `beforeEach`, not repeated
login steps in every test.

```typescript
// fixtures/auth.fixture.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';

type AuthFixtures = {
  authenticatedPage: DashboardPage;
};

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('admin', 'password123');
    await use(new DashboardPage(page));
  },
});
```

If the login flow changes (e.g., adding MFA), update the fixture once — not 500 tests.

### 5.5 Worker-Scoped vs. Test-Scoped Fixtures

| Scope | Created | Use For |
|-------|---------|---------|
| **Test** `({ page })` | Fresh per test | POMs, unique test data, DOM interaction |
| **Worker** `({ browser })` | Once per OS process | DB connections, API auth tokens, global seed data |

```typescript
export const test = base.extend<
  { todoPage: TodoPage },                    // Test-scoped types
  { apiToken: string }                        // Worker-scoped types
>({
  apiToken: [async ({ browser }, use) => {
    const token = await fetchAPIToken();
    await use(token);
  }, { scope: 'worker' }],

  todoPage: async ({ page }, use) => {
    await use(new TodoPage(page));
  },
});
```

**Rule:** Use worker scope for stateless shared resources. Use test scope for anything touching
the DOM. Never share mutable state across tests.

### 5.6 Authentication via Setup Projects

For suite-wide auth state, use Playwright's setup project pattern.

```typescript
// playwright.config.ts (projects section)
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'chromium',
    use: {
      ...devices['Desktop Chrome'],
      storageState: 'playwright/.auth/user.json',
    },
    dependencies: ['setup'],
  },
],
```

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';
const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Username').fill(process.env.TEST_USER!);
  await page.getByLabel('Password').fill(process.env.TEST_PASS!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('/dashboard');
  await page.context().storageState({ path: authFile });
});
```

---

## 6. Network Engineering: Determinism Over Reality

### 6.1 Hybrid Network Strategy

| Category | Backend | When |
|----------|---------|------|
| **E2E smoke tests** | Real | Sanity checks, critical paths |
| **Functional logic tests** | Mocked | Speed, stability, edge case coverage |

### 6.2 API Mocking with `page.route()`

```typescript
// Force a specific error state — no need to find a failing credit card
await page.route('**/api/payment', route => {
  route.fulfill({
    status: 402,
    contentType: 'application/json',
    body: JSON.stringify({ error: 'Insufficient Funds' }),
  });
});

await page.goto('/checkout');
await expect(page.getByRole('alert')).toHaveText('Insufficient Funds');
```

### 6.3 Block Third-Party Dependencies

Never allow tests to fail because a third-party service (analytics, pixels, chat widgets) is
slow or down. Block them globally.

```typescript
// global-setup.ts or in playwright.config.ts use.contextOptions
await page.route('**/*analytics*', route => route.abort());
await page.route('**/*facebook*', route => route.abort());
await page.route('**/*intercom*', route => route.abort());
```

### 6.4 HAR Recording and Replay

Record a "golden path" session and replay it for deterministic frontend regression tests.

```typescript
// Record
await page.routeFromHAR('tests/fixtures/checkout.har', {
  update: true,
  url: '**/api/**',
});

// Replay (in subsequent runs)
await page.routeFromHAR('tests/fixtures/checkout.har', {
  url: '**/api/**',
});
```

---

## 7. Test Isolation

Every test is independent. No shared mutable state. Each test gets a fresh `BrowserContext`.

```typescript
// ❌ NEVER — state leakage prevents parallel execution
let sharedUserId: string;
test('create user', async ({ page }) => { sharedUserId = await createUser(page); });
test('edit user', async ({ page }) => { await editUser(page, sharedUserId); });

// ✅ ALWAYS — self-contained via fixtures
test('edit user', async ({ authenticatedPage, apiToken }) => {
  // Each test creates its own data or receives it from fixtures
});
```

---

## 8. Observability & Debugging

### 8.1 Trace Viewer

The "black box recorder" — captures screenshots per action, DOM snapshots (time travel),
HAR network data, and console logs.

**Configuration standard:** `trace: 'on-first-retry'` — saves resources on passing tests, full
fidelity on failures.

### 8.2 UI Mode

`npx playwright test --ui` — preferred for local development over `--headed`.

Features: Timeline visualization (v1.58), network panel with JSON formatting, step-by-step
time travel.

### 8.3 Console Assertion Policy ("Clean Console")

Use `page.consoleMessages()` (v1.56+) to enforce zero console errors during execution.

```typescript
test('page loads without console errors', async ({ page }) => {
  const errors: string[] = [];
  page.on('console', msg => {
    if (msg.type() === 'error') errors.push(msg.text());
  });

  await page.goto('/dashboard');
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();

  expect(errors, 'Console errors detected during page load').toHaveLength(0);
});
```

### 8.4 Speedboard (v1.57+)

The Speedboard tab in the HTML report identifies consistently slow tests. If a test appears
at the top of the Speedboard, it needs **refactoring** (inefficient locators, excessive polling) —
not a higher timeout.

### 8.5 `test.step()` for Trace Readability

```typescript
test('checkout flow', async ({ page }) => {
  const orderId = await test.step('add item to cart', async () => {
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await expect(page.getByRole('status')).toHaveText('Item added');
    return await page.getByTestId('order-id').textContent();
  });

  await test.step('complete purchase', async () => {
    await page.getByRole('link', { name: 'Cart' }).click();
    await page.getByRole('button', { name: 'Checkout' }).click();
    await expect(page.getByRole('heading')).toHaveText('Order Confirmed');
  });
});
```

---

## 9. TypeScript Standards

### 9.1 The `tsconfig.json` Mandate

The AI must verify or generate a strict configuration:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022"
  }
}
```

- **`noImplicitAny`**: Prevents untyped page objects. Critical for AI inference in future edits.
- **`strictNullChecks`**: Forces explicit handling of nullable returns.

### 9.2 Type-Safe Imports and Definitions

```typescript
// ✅ Use import type for type-only imports
import type { Page, Locator, BrowserContext } from '@playwright/test';
import { test, expect } from '@playwright/test';

// ✅ Type fixture definitions explicitly — enables IntelliSense
type MyFixtures = {
  loginPage: LoginPage;
  apiClient: APIRequestContext;
};

// ✅ Const assertions for test data
const TEST_USERS = {
  admin: { username: 'admin', password: 'admin123' },
  viewer: { username: 'viewer', password: 'viewer123' },
} as const;

// ✅ Strictly typed Page Object constructors
export class MyPage {
  constructor(private readonly page: Page) {}
}

// ❌ NEVER
const data: any = await page.evaluate(() => window.someData);
```

---

## 10. Configuration Reference

### 10.1 `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'blob' : 'html',
  timeout: 30_000,

  expect: {
    timeout: 5_000,
  },

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### 10.2 Config Optimization Matrix

| Option | Recommended Value | Impact |
|--------|-------------------|--------|
| `fullyParallel` | `true` | Maximizes CPU usage across workers |
| `forbidOnly` | `!!process.env.CI` | Fails CI if `.only` is left in code |
| `retries` | `process.env.CI ? 2 : 0` | Safety net in CI; force investigation locally |
| `workers` | `process.env.CI ? 1 : undefined` | Prevents resource starvation in CI containers |
| `reporter` | `process.env.CI ? 'blob' : 'html'` | Enables sharding merge workflow |
| `trace` | `'on-first-retry'` | Full debugging data only on failures |
| `screenshot` | `'only-on-failure'` | Saves disk; captures evidence when needed |
| `video` | `'retain-on-failure'` | Discards passing test videos automatically |

---

## 11. CI/CD — GitHub Actions with Sharding

### 11.1 Complete Workflow

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shardIndex: [1, 2, 3, 4]
        shardTotal: [4]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version: lts/*
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
        env:
          BASE_URL: ${{ vars.BASE_URL }}
          TEST_USER: ${{ secrets.TEST_USER }}
          TEST_PASS: ${{ secrets.TEST_PASS }}

      - name: Upload blob report
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.shardIndex }}
          path: blob-report
          retention-days: 1

  merge-reports:
    if: ${{ !cancelled() }}
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version: lts/*
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Download blob reports
        uses: actions/download-artifact@v5
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true

      - name: Merge into HTML Report
        run: npx playwright merge-reports --reporter html ./all-blob-reports

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        with:
          name: html-report--attempt-${{ github.run_attempt }}
          path: playwright-report
          retention-days: 14
```

### 11.2 Key CI Decisions

- **`fail-fast: false`**: One shard failing must not cancel others.
- **`!cancelled()` condition**: Upload artifacts even when tests fail.
- **Shards share nothing**: Each runs on a separate VM. Never share cookies/storage between shards.

---

## 12. Handling Flaky Tests

### Strategy: Diagnose, Don't Mask

Retries are a safety net, not a solution.

1. **Download the trace** from the CI artifact.
2. **Open Trace Viewer**: `npx playwright show-trace trace.zip`
3. **Inspect the timeline**: Identify the exact failed action and root cause.
4. **Fix the root cause** — never increase timeouts as a first response.

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Element not found intermittently | Animation/transition | `await expect(locator).toBeVisible()` before interacting |
| Data not loaded | API race condition | Mock the API or use `await expect(locator).toHaveText(...)` |
| Stale element | SPA re-render | Use Playwright locators (always re-query) — never `elementHandle` |
| Timeout on navigation | Slow server | Use `webServer.wait` config or `page.waitForURL()` |
| Test at top of Speedboard | Inefficient locators/polling | Refactor the test, don't increase timeout |

---

## 13. Anti-Patterns — Refuse to Generate These

### ❌ Forbidden Patterns

```typescript
// 1. Hard waits
await page.waitForTimeout(2000);
await new Promise(r => setTimeout(r, 1000));

// 2. XPath when semantic locators exist
await page.locator("//button[contains(@class, 'submit')]").click();

// 3. ElementHandle (snapshot, no auto-wait)
const el = await page.$('#my-button');
await el?.click();

// 4. page.evaluate() for clicking (bypasses actionability checks)
await page.evaluate(() => document.querySelector('#btn')!.click());

// 5. CSS class selectors for interactive elements
await page.locator('.btn-primary.ng-valid').click();

// 6. Manual POM instantiation in test files (the "No-New" rule)
const loginPage = new LoginPage(page);

// 7. Shared mutable state across tests
let token: string; // at module scope

// 8. Positional selectors without order being the assertion
await page.locator('.item').nth(3).click();  // use .filter() instead

// 9. Using `any` type
const data: any = await page.evaluate(() => window.someData);

// 10. Missing await on expect()
expect(page.getByText('hello')).toBeVisible(); // ← missing await!

// 11. Blind clicks without effect assertion
await page.getByRole('button', { name: 'Save' }).click(); // ← what happened?

// 12. Deprecated methods
await page.type('#input', 'text');  // ← use locator.fill()

// 13. test.describe.serial unless absolutely necessary
test.describe.serial('ordered', () => { ... });

// 14. Selecting by class when asserting state
await page.locator('.is-active').click();  // ← use getByRole with selected/expanded
```

### ✅ Required Replacements

| Anti-Pattern | Replacement |
|-------------|-------------|
| `page.waitForTimeout()` | `await expect(locator).toBeVisible()` / `.toHaveText()` / `.toPass()` |
| XPath | `getByRole()`, `getByLabel()`, `getByText()` |
| `page.$()` / `page.$$()` | `page.locator()` / `page.getByRole()` |
| `page.evaluate(() => el.click())` | `locator.click()` |
| CSS class selectors | User-facing locators |
| `new PageObject(page)` in tests | Custom fixtures via `test.extend()` |
| `page.type()` | `locator.fill()` or `locator.pressSequentially()` |
| `.nth(n)` for disambiguation | `.filter({ hasText })` or `.filter({ has })` |
| `test.describe.serial()` | Independent tests with proper fixtures |
| `any` type | Proper TypeScript types |
| Blind click | Click + assert the resulting state change |
| `locator('.is-active')` | `getByRole('tab', { selected: true })` or assert with `toContainClass()` |

---

## 14. Assertions Reference

### Polling (Web-First) Assertions — Use for DOM State

```typescript
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeAttached();       // In DOM, even if hidden
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();
await expect(locator).toBeChecked();
await expect(locator).toHaveText('exact');
await expect(locator).toContainText('partial');
await expect(locator).toHaveAttribute('href', '/home');
await expect(locator).toHaveValue('input value');
await expect(locator).toHaveCount(3);        // Wait for list to have n items
await expect(locator).toHaveClass(/active/);
await expect(locator).toContainClass('active');  // v1.52+
await expect(locator).toHaveCSS('color', 'rgb(0, 0, 0)');
await expect(page).toHaveURL(/dashboard/);
await expect(page).toHaveTitle('My App');
await expect(locator).toMatchAriaSnapshot(`
  - heading "Dashboard"
  - navigation:
    - link "Home"
    - link "Settings"
`);

// Retry an entire block
await expect(async () => {
  await page.getByRole('button', { name: 'Refresh' }).click();
  await expect(page.getByText('Updated')).toBeVisible();
}).toPass({ timeout: 15_000 });
```

### Non-Polling Assertions — Use for Static Values Only

```typescript
// These do NOT retry — use only on already-resolved values
expect(await page.title()).toBe('My App');
expect(responseData.length).toBeGreaterThan(0);
```

---

## 15. Project Structure

```
project-root/
├── playwright.config.ts
├── tsconfig.json                # strict: true
├── package.json
├── tests/
│   ├── auth.setup.ts
│   ├── login.spec.ts
│   ├── dashboard.spec.ts
│   └── checkout.spec.ts
├── pages/
│   ├── login.page.ts
│   ├── dashboard.page.ts
│   └── checkout.page.ts
├── fixtures/
│   ├── index.ts                 # Main test export with all fixtures
│   └── auth.fixture.ts          # Auth-specific fixtures
├── utils/
│   └── test-data.ts
├── playwright/
│   └── .auth/                   # gitignored — stored auth state
├── blob-report/                 # gitignored — CI blob output
└── playwright-report/           # gitignored — merged HTML report
```

---

## 16. Quick Reference — Commands

```bash
npx playwright test                          # Run all tests
npx playwright test --ui                     # UI mode (preferred for local dev)
npx playwright test tests/login.spec.ts      # Run specific file
npx playwright test tests/login.spec.ts:15 --debug  # Debug specific test
npx playwright test --project=chromium       # Run specific project
npx playwright codegen http://localhost:3000 # Generate test code
npx playwright show-report                   # View last HTML report
npx playwright show-trace trace.zip          # View a trace file
npx playwright test --update-snapshots       # Update aria/visual snapshots
npx playwright init-agents --loop=claude     # Generate AI agent definitions
```
