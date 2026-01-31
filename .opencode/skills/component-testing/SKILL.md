---
name: component-testing
description: Vitest patterns for testing React components. Use when writing unit tests for components, hooks, or utilities.
---

# Component Testing with Vitest

This skill covers patterns for testing React components using Vitest and the mock-server.

## Core Philosophy

> **The frontend responds to whatever the server sends—never assume server data shape.**

Test **UI responses**, not server behavior. We test how the app displays errors, loading states, and empty states.

---

## Mock Server (NOT MSW)

> **Use mock-server scenarios, NOT MSW or vi.mock() for API mocking**

The mock-server runs at `http://localhost:3002` during tests.

### Switching Scenarios

```typescript
// Set a specific scenario before the test
await fetch('http://localhost:3002/_scenario', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'getTeam', scope: 'success', scenario: 'default' })
});

// Set error scenario
await fetch('http://localhost:3002/_scenario', {
  method: 'POST',
  body: JSON.stringify({ name: 'getTeam', scope: 'notFound' })
});

// Reset all routes between tests
await fetch('http://localhost:3002/_reset', { method: 'POST' });
```

### Anti-Pattern: Hook-Level Mocking

```typescript
// WRONG - mocking at hook level
vi.mock('../api/teams', () => ({ useTeam: () => ({ error: true }) }));

// CORRECT - use mock-server scenario
await fetch('http://localhost:3002/_scenario', {
  method: 'POST',
  body: JSON.stringify({ name: 'getTeam', scope: 'error' })
});
```

---

## Test Categories

Every API call should have tests for:

| Scenario | Status | What to Test |
|----------|--------|--------------|
| Success | 200 | Happy path, expected data renders |
| Empty | 200 | Empty arrays, null values handled |
| Not Found | 404 | Resource missing, error UI shown |
| Unauthorized | 401 | Token expired, redirect to login |
| Forbidden | 403 | No permission, access denied UI |
| Server Error | 500 | Backend failure, error boundary |

---

## Test Structure

### Basic Component Test

```typescript
import { render, screen } from '@testing-library/react';
import { TeamCard } from './TeamCard';

describe('TeamCard', () => {
  beforeEach(async () => {
    // Reset mock server
    await fetch('http://localhost:3002/_reset', { method: 'POST' });
  });

  it('displays team name and record', async () => {
    // Arrange - set up scenario
    await fetch('http://localhost:3002/_scenario', {
      method: 'POST',
      body: JSON.stringify({ name: 'getTeam', scenario: 'default' })
    });

    // Act
    render(<TeamCard teamId={1} />);

    // Assert
    expect(await screen.findByText('Chicago Bears')).toBeInTheDocument();
    expect(screen.getByText('8-4')).toBeInTheDocument();
  });

  it('shows loading state', () => {
    render(<TeamCard teamId={1} />);

    expect(screen.getByTestId('loading-skeleton')).toBeInTheDocument();
  });

  it('handles 404 error', async () => {
    await fetch('http://localhost:3002/_scenario', {
      method: 'POST',
      body: JSON.stringify({ name: 'getTeam', scope: 'notFound' })
    });

    render(<TeamCard teamId={999} />);

    expect(await screen.findByText('Team not found')).toBeInTheDocument();
  });
});
```

### Testing User Interactions

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

it('submits form with correct data', async () => {
  const user = userEvent.setup();

  render(<PlayerForm />);

  await user.type(screen.getByLabelText('Name'), 'John Doe');
  await user.selectOptions(screen.getByLabelText('Position'), 'QB');
  await user.click(screen.getByRole('button', { name: 'Save' }));

  expect(await screen.findByText('Player saved')).toBeInTheDocument();
});
```

---

## Common Patterns

### Testing Loading States

```typescript
it('shows skeleton while loading', () => {
  render(<PlayerList />);

  expect(screen.getAllByTestId('skeleton-row')).toHaveLength(5);
});
```

### Testing Empty States

```typescript
it('shows empty message when no players', async () => {
  await fetch('http://localhost:3002/_scenario', {
    method: 'POST',
    body: JSON.stringify({ name: 'getPlayers', scenario: 'empty' })
  });

  render(<PlayerList />);

  expect(await screen.findByText('No players found')).toBeInTheDocument();
});
```

### Testing Error States

```typescript
it('shows error boundary on server error', async () => {
  await fetch('http://localhost:3002/_scenario', {
    method: 'POST',
    body: JSON.stringify({ name: 'getPlayers', scope: 'error' })
  });

  render(<PlayerList />);

  expect(await screen.findByText('Something went wrong')).toBeInTheDocument();
  expect(screen.getByRole('button', { name: 'Retry' })).toBeInTheDocument();
});
```

### Testing Auth Redirects

```typescript
it('redirects to login on 401', async () => {
  const navigate = vi.fn();
  vi.mock('react-router-dom', () => ({
    useNavigate: () => navigate
  }));

  await fetch('http://localhost:3002/_scenario', {
    method: 'POST',
    body: JSON.stringify({ name: 'getProfile', scope: 'unauthorized' })
  });

  render(<ProfilePage />);

  await waitFor(() => {
    expect(navigate).toHaveBeenCalledWith('/login');
  });
});
```

---

## File Organization

```
src/
├── components/
│   ├── TeamCard/
│   │   ├── TeamCard.tsx
│   │   ├── TeamCard.test.tsx    # Component tests
│   │   └── index.ts
│   └── PlayerList/
│       ├── PlayerList.tsx
│       ├── PlayerList.test.tsx
│       └── index.ts
└── hooks/
    ├── useTeam.ts
    └── useTeam.test.ts          # Hook tests
```

---

## Running Tests

```bash
# Run all tests
npm test

# Run specific file
npm test -- TeamCard.test.tsx

# Run with coverage
npm test -- --coverage

# Watch mode
npm test -- --watch
```

---

## Test Coverage Requirements

> **Write comprehensive tests from the start. Every scenario, every edge case.**

### Scenario Coverage Matrix (Verify for EVERY component)

| Scenario | Status | What to Verify |
|----------|--------|----------------|
| Success (200) | | Data renders correctly |
| Empty (200) | | Empty arrays, null values handled |
| Not Found (404) | | Resource missing UI shown |
| Unauthorized (401) | | Redirect to login |
| Forbidden (403) | | Access denied UI shown |
| Server Error (500) | | Error boundary triggered |
| Loading | | Skeleton/spinner visible |
| Slow Network | | Loading persists appropriately |

### User Interaction Coverage (Test these as you write)

- [ ] All clickable elements have click tests
- [ ] Form submission with valid data
- [ ] Form submission with invalid data
- [ ] Keyboard navigation (Tab, Enter, Escape)
- [ ] Edge cases (double-click, rapid clicks)

### Component State Coverage (Each state needs a test)

- [ ] Initial render state
- [ ] After data loads
- [ ] After user interaction
- [ ] After error occurs
- [ ] After retry/recovery

### Accessibility Testing (Verify for EVERY component)

- [ ] ARIA attributes verified with assertions
- [ ] Focus management correct after interactions
- [ ] Screen reader text present for dynamic content
- [ ] Keyboard-only usability confirmed

### Edge Cases (Include in EVERY test file)

- [ ] Very long text (truncation works)
- [ ] Very large numbers (formatting correct)
- [ ] Special characters in data (renders safely)
- [ ] Missing optional fields (doesn't crash)
- [ ] Timezone-sensitive dates (handles correctly)

---

## React Testing Best Practices (Apply as you write)

### Testing Library Usage

| Guideline | Check |
|-----------|-------|
| **Query priority** | Using getByRole over getByTestId where possible |
| **User-centric** | Queries match how users find elements |
| **Implementation-agnostic** | Not testing internal state or props |
| **Interactions** | Using userEvent over fireEvent |

### Async Testing Patterns

```typescript
// GOOD - findBy* for async elements
expect(await screen.findByText('Loaded')).toBeInTheDocument();

// GOOD - waitFor for state changes
await waitFor(() => {
  expect(mockFn).toHaveBeenCalled();
});

// BAD - arbitrary delays
await new Promise(r => setTimeout(r, 1000));  // NEVER DO THIS
```

### Test Quality (Self-check as you write)

- [ ] Tests isolated (no shared state between tests)
- [ ] Assertions meaningful (not just "renders without crashing")
- [ ] Mock server reset in beforeEach
- [ ] Async operations properly awaited
- [ ] No flaky tests (run 10x to verify)

---

## Checklist Before PR

- [ ] All scenarios tested (success, empty, 404, 401, 403, 500)
- [ ] Loading states tested
- [ ] User interactions tested with userEvent
- [ ] Mock server scenarios used (not vi.mock for API)
- [ ] Accessibility assertions included
- [ ] Edge cases covered
- [ ] `npm run lint && npm test` passes
