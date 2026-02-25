---
name: pwa-e2e-testing
description: Guide for writing reliable Playwright E2E tests for mobile-first Progressive Web Apps. Use when creating tests, debugging flaky tests, handling photo uploads without camera hardware, navigation timing, or form submissions. Covers test-utils helpers, pre-authentication, parallel isolation, and mobile device testing.
---

# Mobile App E2E Testing with Playwright

## Purpose

Guidelines for writing reliable E2E tests for mobile-first apps using Playwright. Covers timing issues, parallel test isolation, photo uploads without camera hardware, and mobile testing patterns.

## CRITICAL: Test-Utils Helpers (USE THESE)

**Before writing any test code, import helpers from `e2e/test-utils.ts`:**

```typescript
import {
   navigateToWorkflow,
   createEntity,
   submitForm,
   selectEntity,
} from '../test-utils';

test('my workflow test', async ({ page }) => {
   await navigateToWorkflow(page, 'create');
   await createEntity(page, `E2E-${Date.now()}`);
});
```

**DO NOT** copy navigation/form logic into tests. The helpers handle:
- Navigation race conditions (workflow pickers may appear unexpectedly)
- Form submission timing issues
- React rendering delays

## Core Testing Patterns

### Pre-Authentication Setup

Use global setup to authenticate once and reuse credentials:

```typescript
// e2e/global-setup.ts
export default async function globalSetup() {
   const browser = await chromium.launch();
   const context = await browser.newContext();
   const page = await context.newPage();

   await page.goto('/');
   await page.getByPlaceholder('Email address').fill(email);
   await page.getByPlaceholder('Password').fill(password);
   await page.getByRole('button', { name: 'Sign In' }).click();

   await page.context().storageState({ path: authFile });
}
```

### Timing & Race Conditions

**Wait for page load:**

```typescript
await page.goto('/workflow');
await page.waitForLoadState('domcontentloaded');
await expect(
   page.getByRole('heading', { name: 'My Workflow' }),
).toBeVisible({ timeout: 10000 });
```

**Wait for forms to render before interaction:**

```typescript
await page.getByRole('button', { name: 'Create' }).click();
await expect(
   page.getByRole('heading', { name: 'Create Record' }),
).toBeVisible();
// Now safe to interact with form fields
```

**Wait for search results to stabilize:**

```typescript
await page.getByRole('textbox', { name: 'Search...' }).fill(searchTerm);
await page.waitForTimeout(500);
await page.getByRole('button', { name: new RegExp(searchTerm) }).first().click();
```

### Parallel Test Isolation

**Problem:** Tests running in parallel can create records with identical identifiers.

**Solution:** Use timestamps for unique IDs and `.first()` to select the newest match:

```typescript
const TEST_ID = `E2E-TEST-${Date.now()}`;

// After creating, use .first() to handle duplicates from parallel tests
await page.getByRole('button', { name: new RegExp(TEST_ID) }).first().click();
```

### Photo Upload Without Camera Hardware

**Problem:** Headless browsers lack camera access. Photo capture components show an error modal.

**Solution:** Use the file chooser API:

```typescript
// 1. Click button that triggers photo capture
await page.getByRole('button', { name: 'Take Photo' }).click();
await page.waitForTimeout(500);

// 2. Camera fails — use file chooser
const [fileChooser] = await Promise.all([
   page.waitForEvent('filechooser'),
   page.getByRole('button', { name: 'Choose File' }).click(),
]);

// 3. Upload fixture image
await fileChooser.setFiles('e2e/fixtures/test-photo.png');
await page.waitForTimeout(500);

// 4. Wait for upload to complete
await expect(page.getByText('Uploading...')).toBeHidden({ timeout: 60000 });
await page.waitForTimeout(500); // Let DB write finish

// 5. Verify image loaded
const img = page.locator('img[alt="Photo"]');
await expect(img).toBeVisible({ timeout: 15000 });
```

**CRITICAL: Background uploads need time to persist.** Photo uploads run in the background — the storage upload completes, then a DB record is saved asynchronously. Always add a 500ms wait after confirming upload completion before navigating away.

### Form Interaction Patterns

**Combobox selection (multiple on same page):**

```typescript
await page.getByRole('combobox').nth(0).selectOption('Option A');
await page.getByRole('combobox').nth(1).selectOption('Option B');
```

**Scroll before click:**

```typescript
const button = page.getByRole('button', { name: 'Add' }).first();
await button.scrollIntoViewIfNeeded();
await button.click();
```

### Multi-Step Workflow Navigation

```typescript
const steps = ['Step A', 'Step B', 'Step C'];

for (const step of steps) {
   await expect(page.getByRole('heading', { name: step })).toBeVisible();
   await page.getByRole('button', { name: 'Pass' }).click();
}
```

### Mobile Testing

```typescript
// playwright.config.ts
projects: [
   { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
   {
      name: 'Mobile Chrome',
      use: {
         ...devices['Pixel 5'],
         deviceScaleFactor: 1,
         permissions: ['geolocation', 'camera'],
         geolocation: { latitude: 40.7128, longitude: -74.006 },
      },
   },
];
```

## Playwright Configuration

```typescript
export default defineConfig({
   testDir: './e2e',
   timeout: 30000,
   expect: { timeout: 5000 },
   fullyParallel: true,
   workers: process.env.CI ? 1 : 6,

   use: {
      baseURL: 'http://localhost:5173',
      trace: 'on-first-retry',
      screenshot: 'only-on-failure',
   },

   globalSetup: require.resolve('./e2e/global-setup.ts'),

   projects: [
      { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
      {
         name: 'Mobile Chrome',
         use: { ...devices['Pixel 5'], deviceScaleFactor: 1 },
      },
      {
         name: 'chromium-unauthenticated',
         use: { ...devices['Desktop Chrome'] },
      },
   ],
});
```

## Test File Organization

```
apps/app/
├── e2e/
│   ├── fixtures/              # Test images
│   ├── global-setup.ts        # Auth setup
│   ├── test-utils.ts          # Shared helpers
│   ├── app.spec.ts            # General app tests
│   └── tests/
│       ├── auth/              # Authentication tests
│       ├── workflow-a/        # Workflow A tests
│       └── workflow-b/        # Workflow B tests
├── playwright.config.ts
└── test-plans/
    └── test-plan.md
```

## Common Issues & Solutions

### Element detached from DOM

React re-renders between locating and clicking. Use atomic operations:

```typescript
// BAD
const button = page.getByRole('button', { name: 'Submit' });
await button.click();

// GOOD
await page.getByRole('button', { name: 'Submit' }).click();
```

### Test passes locally, fails in CI

- Increase timeouts in CI
- Reduce workers: `workers: process.env.CI ? 1 : 6`
- Add explicit waits before critical interactions

### Photo resume tests can't find uploaded photos

The test navigated away before background upload persisted to DB. Always wait after upload completes:

```typescript
await page.waitForTimeout(500); // Let DB write finish
// NOW safe to navigate away
```

### Button click doesn't trigger React handler

1. Check visibility: `await expect(button).toBeEnabled()`
2. Add delay: `await page.waitForTimeout(500)`
3. Try force click: `await button.click({ force: true })`
4. Try JS click: `await button.evaluate(b => b.click())`
5. Check for overlays blocking the button

## Best Practices

- **Avoid deprecated APIs**: Don't use `{ waitUntil: 'networkidle' }` — use explicit waits
- **Prefer role-based selectors**: `getByRole('button', { name: 'Submit' })` over CSS selectors
- **Use `--reporter=list`** in automated runs (default HTML reporter blocks the terminal)
- **Use `test.slow()`** for photo upload tests (3x timeout)
- **Mark blocking tests**: Use `test.fixme()` for tests blocked by known bugs

## Debugging Checklist

1. Read the error message — what was expected vs actual?
2. Check screenshots/traces in `test-output/`
3. Check if page loaded — is the expected heading visible?
4. Check for overlays — modals, toasts, error banners
5. Check console errors and network failures
6. Add delays if timing-related
7. Try force click if button isn't responding
8. Run headed: `--headed --slowmo=500`
9. Isolate the test — run just that one file
10. Check if it's a parallel execution issue
