---
name: e2e-testing
description: Playwright patterns for end-to-end testing in React applications. Use when writing E2E tests for user flows, pages, or integrations.
---

# E2E Testing with Playwright

This skill covers patterns for end-to-end testing using Playwright and the mock-server.

## Setup

E2E tests use the mock-server at `http://localhost:3001`.

### Configuration

`playwright.config.ts`:
```typescript
export default defineConfig({
  testDir: './e2e',
  webServer: {
    command: 'npm run dev:mock',
    port: 5173,
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Running Tests

```bash
# Run all E2E tests
npm run test:e2e

# Run specific test file
npx playwright test e2e/roster.spec.ts

# Run with UI mode (debugging)
npx playwright test --ui

# Run headed (see browser)
npx playwright test --headed

# Generate report
npx playwright show-report
```

---

## Test Structure

### Basic Page Test

```typescript
import { test, expect } from '@playwright/test';

test.describe('Roster Page', () => {
  test.beforeEach(async ({ page }) => {
    // Reset mock server state
    await page.request.post('http://localhost:3001/_reset');
  });

  test('displays player list', async ({ page }) => {
    await page.goto('/roster');

    await expect(page.getByRole('heading', { name: 'Roster' })).toBeVisible();
    await expect(page.getByTestId('player-row')).toHaveCount(53);
  });

  test('filters by position', async ({ page }) => {
    await page.goto('/roster');

    await page.getByRole('combobox', { name: 'Position' }).selectOption('QB');

    await expect(page.getByTestId('player-row')).toHaveCount(3);
    await expect(page.getByText('Quarterback')).toBeVisible();
  });
});
```

### Testing User Flows

```typescript
test('complete trade flow', async ({ page }) => {
  // Navigate to trade page
  await page.goto('/trade');

  // Select trade partner
  await page.getByRole('button', { name: 'Select Team' }).click();
  await page.getByText('Green Bay Packers').click();

  // Add player to offer
  await page.getByTestId('add-player-btn').click();
  await page.getByText('J. Williams').click();

  // Submit trade
  await page.getByRole('button', { name: 'Propose Trade' }).click();

  // Verify confirmation
  await expect(page.getByText('Trade proposed successfully')).toBeVisible();
});
```

---

## Mock Server Control

### Setting Scenarios

```typescript
test('handles API error gracefully', async ({ page }) => {
  // Set error scenario before navigation
  await page.request.post('http://localhost:3001/_scenario', {
    data: { name: 'getRoster', scope: 'error' }
  });

  await page.goto('/roster');

  await expect(page.getByText('Failed to load roster')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Retry' })).toBeVisible();
});
```

### Using Presets

```typescript
test('new user sees empty state', async ({ page }) => {
  await page.request.post('http://localhost:3001/_preset', {
    data: { name: 'new-user' }
  });

  await page.goto('/dashboard');

  await expect(page.getByText('Welcome! Create your first league')).toBeVisible();
});
```

---

## Common Patterns

### Authentication

```typescript
test.describe('authenticated routes', () => {
  test.beforeEach(async ({ page }) => {
    // Set authenticated state
    await page.request.post('http://localhost:3001/_preset', {
      data: { name: 'authenticated' }
    });
  });

  test('accesses profile page', async ({ page }) => {
    await page.goto('/profile');
    await expect(page.getByText('Your Profile')).toBeVisible();
  });
});
```

### Form Submission

```typescript
test('creates new player', async ({ page }) => {
  await page.goto('/players/new');

  await page.getByLabel('First Name').fill('John');
  await page.getByLabel('Last Name').fill('Doe');
  await page.getByLabel('Position').selectOption('QB');
  await page.getByLabel('Overall').fill('75');

  await page.getByRole('button', { name: 'Create Player' }).click();

  await expect(page).toHaveURL(/\/players\/\d+/);
  await expect(page.getByText('John Doe')).toBeVisible();
});
```

### Table Interactions

```typescript
test('sorts standings by wins', async ({ page }) => {
  await page.goto('/standings');

  // Click header to sort
  await page.getByRole('columnheader', { name: 'W' }).click();

  // Verify first row has most wins
  const firstRow = page.getByTestId('standings-row').first();
  await expect(firstRow.getByTestId('wins')).toHaveText('12');
});
```

### Navigation

```typescript
test('navigates through tabs', async ({ page }) => {
  await page.goto('/team/1');

  await page.getByRole('tab', { name: 'Roster' }).click();
  await expect(page.getByTestId('roster-table')).toBeVisible();

  await page.getByRole('tab', { name: 'Schedule' }).click();
  await expect(page.getByTestId('schedule-grid')).toBeVisible();

  await page.getByRole('tab', { name: 'Stats' }).click();
  await expect(page.getByTestId('stats-panel')).toBeVisible();
});
```

---

## Page Objects (Optional)

For complex pages, use page objects:

```typescript
// e2e/pages/RosterPage.ts
export class RosterPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/roster');
  }

  async filterByPosition(position: string) {
    await this.page.getByRole('combobox', { name: 'Position' }).selectOption(position);
  }

  async getPlayerCount() {
    return this.page.getByTestId('player-row').count();
  }

  async clickPlayer(name: string) {
    await this.page.getByText(name).click();
  }
}

// e2e/roster.spec.ts
test('filters roster', async ({ page }) => {
  const roster = new RosterPage(page);
  await roster.goto();
  await roster.filterByPosition('WR');
  expect(await roster.getPlayerCount()).toBe(6);
});
```

---

## Visual Testing

```typescript
test('roster page matches snapshot', async ({ page }) => {
  await page.goto('/roster');
  await expect(page).toHaveScreenshot('roster-page.png');
});
```

---

## Debugging Tips

### Use UI Mode

```bash
npx playwright test --ui
```

Interactive debugging with time-travel, DOM inspection, and step-through.

### Pause on Failure

```typescript
test('debug this test', async ({ page }) => {
  await page.goto('/roster');
  await page.pause(); // Opens inspector
  // ... rest of test
});
```

### Trace Viewer

```bash
npx playwright test --trace on
npx playwright show-report
```

Click on failed tests to see trace with screenshots, network, and console.

---

## File Organization

```
e2e/
├── pages/              # Page objects (optional)
│   ├── RosterPage.ts
│   └── TradePage.ts
├── fixtures/           # Custom fixtures
│   └── auth.ts
├── roster.spec.ts      # Roster page tests
├── trade.spec.ts       # Trade flow tests
├── auth.spec.ts        # Authentication tests
└── navigation.spec.ts  # Navigation tests
```

---

## Test Coverage Requirements

> **Write comprehensive E2E tests from the start. Every user journey, every edge case.**

### User Flow Coverage Matrix (Verify for EVERY feature)

| Flow | Happy Path | Error Path | Status |
|------|------------|------------|--------|
| Authentication | Login, logout | Invalid creds, expired session | |
| Navigation | All routes accessible | 404, unauthorized routes | |
| Data CRUD | Create, read, update, delete | Validation errors, conflicts | |
| Form Submission | Valid data submits | Invalid data, server errors | |
| Search/Filter | Basic search works | No results, special chars | |

### Critical User Journeys (Must test these)

- [ ] New user onboarding flow (first login to first action)
- [ ] Core feature happy paths (the main thing users do)
- [ ] Error recovery flows (what happens when things fail)
- [ ] Session timeout handling (mid-flow expiration)

### Cross-Browser/Device Testing (Required coverage)

| Environment | Tested? |
|-------------|---------|
| Desktop Chrome | |
| Mobile viewport (375px) | |
| Tablet viewport (768px) | |
| Touch interactions | |

### Edge Cases (Include in EVERY test suite)

- [ ] Session expiry mid-flow
- [ ] Network disconnection handling
- [ ] Slow network conditions (3G throttling)
- [ ] Concurrent tab usage
- [ ] Browser back/forward navigation
- [ ] Page refresh mid-flow (state preserved?)

---

## UX Validation Requirements (Verify as you write tests)

### Accessibility in Real Flows

- [ ] Can complete all flows with keyboard only
- [ ] Screen reader announces state changes
- [ ] Focus managed correctly through multi-step flows
- [ ] Error messages associated with their inputs

### Mobile Usability

- [ ] Touch targets minimum 44x44px
- [ ] Forms usable on small screens (no horizontal scroll)
- [ ] No content hidden off-screen
- [ ] Scrolling behavior correct (no scroll traps)

### Error UX

- [ ] Error messages clear and actionable (not "Error occurred")
- [ ] Form errors inline, not just toast notifications
- [ ] Recovery paths obvious (retry button, help link)
- [ ] No dead ends (always a way forward)

---

## Test Reliability Requirements (Apply as you write)

| Requirement | Check |
|-------------|-------|
| **Independence** | Tests don't depend on other tests running first |
| **No sleeps** | Using `waitFor` and `expect`, not arbitrary delays |
| **Resilient selectors** | Using roles/labels, not brittle CSS selectors |
| **State management** | Mock server reset in beforeEach |

### Anti-Patterns to Avoid

```typescript
// BAD - arbitrary sleep
await page.waitForTimeout(2000);

// GOOD - wait for specific condition
await expect(page.getByText('Loaded')).toBeVisible();

// BAD - brittle selector
await page.locator('.btn-primary-lg').click();

// GOOD - semantic selector
await page.getByRole('button', { name: 'Submit' }).click();
```

---

## Checklist Before PR

- [ ] Tests reset mock server in beforeEach
- [ ] Error scenarios tested (404, 500, network failures)
- [ ] User flows complete (not just happy path)
- [ ] Mobile viewport tested for responsive features
- [ ] Accessibility flows verified (keyboard navigation)
- [ ] Edge cases covered (back button, refresh, timeout)
- [ ] No arbitrary sleeps (all waits are condition-based)
- [ ] `npm run test:e2e` passes
