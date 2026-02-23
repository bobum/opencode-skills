---
name: testing-workflow
description: Comprehensive testing workflow for multi-repository projects. Use when implementing features, fixing bugs, creating PRs, or running test suites.
---

# Testing Workflow

> Use when: implementing features, fixing bugs, creating PRs, or running test suites.

## Core Rule

⛔ **ALL tests MUST pass locally BEFORE creating a PR** ⛔

Every feature, bug fix, or change requires comprehensive tests. Run the full test suite locally and verify all tests pass before pushing.

## Identifying Test Commands

Check the project for test configuration:

| Stack | Common Commands | Config File |
|-------|----------------|-------------|
| .NET | `dotnet test` | `*.csproj`, `*.sln` |
| Node.js | `npm test`, `npm run lint` | `package.json` |
| Python | `pytest`, `python -m pytest` | `pyproject.toml`, `setup.cfg` |
| Go | `go test ./...` | `go.mod` |
| Rust | `cargo test` | `Cargo.toml` |

If the project has a test command table in its AGENTS.md or README, follow that.

## Testing by Stack

### .NET Projects

```bash
dotnet test
```

- **Unit tests**: Service and repository logic
- **Controller tests**: Mock dependencies, test authorization (401/403), error responses
- **Integration tests**: Full stack with test database, test actual SQL queries
- Register new services in test fixtures for integration tests
- Use fixed random seeds for deterministic results where applicable

### Frontend Projects (Node.js/TypeScript)

```bash
npm run lint    # Fix lint errors first
npm test        # Run test suite
```

- **Component tests**: Render, user interactions, state changes
- **Error scenarios**: Test 401 (unauthorized), 403 (forbidden), 404 (not found), 500 (server error)
- **Empty states**: Test what happens when API returns empty arrays
- **Loading states**: Test loading indicators appear
- **Mock APIs**: Use mock server routes or MSW — check project conventions

### Library/Package Projects

```bash
dotnet test     # .NET libraries
npm test        # Node.js packages
pytest          # Python packages
```

- Pure unit tests — no external dependencies
- Test edge cases and boundary conditions
- Use deterministic inputs (fixed seeds, known data sets)

## On Task Completion (MANDATORY)

Before marking any implementation task complete, **run the full test suite** for the affected repository.

**This is non-negotiable.** Do not:
- Mark a task complete without running tests
- Push commits without verifying the full suite passes
- Assume "my changes are isolated" — run the full suite anyway

If tests fail, fix them before considering the task done.

## Pre-PR Checklist

Before creating ANY pull request:

1. ✅ All tests pass with 0 failures
2. ✅ Linting passes with 0 errors (if applicable)
3. ✅ New features have corresponding tests
4. ✅ Error scenarios are tested (not just happy path)
5. ✅ DI/fixture setup is updated if new services were added
6. ✅ Tests are deterministic — no flaky tests, no random seeds without fixed values

## Test Requirements Summary

| Project Type | Test Types | Key Requirements |
|-------------|-----------|-----------------|
| **API/Backend** | Unit + Integration | Controller tests, service tests, integration tests against test DB |
| **Frontend** | Unit + Functional | Component tests, mock API scenarios, error handling tests |
| **Library/Engine** | Unit | Comprehensive unit tests, edge cases, deterministic inputs |

## Dealing with Test Failures

1. **Your changes caused it** → Fix immediately before proceeding
2. **Pre-existing failure** → Note it, do NOT fix as part of your task unless asked
3. **Flaky test** → Investigate root cause (usually timing, randomness, or external dependency). Report to user.

## Multi-Repository Projects

When working across multiple repositories:
- Run tests in **each affected repo** independently
- Test in dependency order (libraries first, then consumers)
- A change in a library repo may require re-running tests in consuming repos
