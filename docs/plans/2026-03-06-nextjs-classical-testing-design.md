# Next.js Classical Testing Skill Design

**Date:** 2026-03-06
**Status:** Approved

## Goal

Create a React/Next.js-specific testing skill based on classical (Boston/Detroit school) testing principles, extending the framework-agnostic `classical-testing` skill. Covers component rendering contracts, user interactions, hooks, Server Actions, Route Handlers, async Server Components, MSW, and sociable test patterns.

## Key Decisions

- **Scope:** Broad — rendering contracts + interactions + hooks + Server Actions + Route Handlers + async RSC workarounds + MSW + sociable tests
- **Interaction testing:** Allowed via RTL (click → assert DOM changed). This IS blackbox testing since we verify observable output, not handler calls.
- **Framework:** Framework-agnostic patterns with Vitest examples, note Jest equivalents where they differ.
- **Relationship:** References `classical-testing` as REQUIRED BACKGROUND. No duplication of Khorikov's 4 quadrants, mock boundaries, or sut() pattern.

## Skill Structure

### 1. Overview
- RTL aligns with classical testing by design (blackbox, state-based verification)
- Reference `classical-testing` for general principles

### 2. What to Test vs Not (Next.js-specific quadrant mapping)
- Client Components → rendering contracts + interactions
- Hooks → output-based via renderHook (shared) or through component (single-use)
- Server Actions → plain async function tests (controllers in Khorikov's model)
- Route Handlers → direct import, NextRequest construction
- Async Server Components → workarounds or E2E
- Full user flows → E2E (Playwright/Cypress)

### 3. Component Rendering Contracts — `cut()` Factory
- `cut()` calls `render()` with defaultProps + overrides
- Uses RTL `screen` queries (getByRole > getByText > getByTestId)
- No beforeEach/afterEach — each test calls `cut()` directly
- Renders full component tree (no shallow rendering)

### 4. User Interaction Testing
- `userEvent.setup()` over `fireEvent`
- Assert on DOM changes, not handler/state calls
- Blackbox boundary table: what's observable vs implementation detail

### 5. Hooks Testing
- `renderHook` with `sut()` pattern for shared hooks
- Test through component when hook is single-use
- Decision table: when to test hook directly vs through component

### 6. Server Actions Testing
- Test as plain async functions
- Mock only Next.js primitives (next/headers, next/navigation, next/cache)
- Same "2 integration tests per use case" rule from classical-testing
- `sut()` factory constructs FormData with overrides

### 7. Route Handler Testing
- Direct import of GET/POST/etc from route.ts
- Construct NextRequest objects in `sut()` factory
- Assert on Response status + JSON body

### 8. Async Server Components
- Three workarounds in preference order:
  1. Direct await (leaf components)
  2. Suspense wrapper (React 19+)
  3. Custom renderAsync helper (nested async trees)
- When to defer to E2E
- Note: evolving area, prefer native support when available

### 9. MSW for Network Boundary Mocking
- `setupServer` with handlers — intercepts at HTTP level
- Why MSW over vi.mock('fetch'): classical = mock at boundary, not import level
- Per-test overrides via `server.use()`
- beforeAll/afterAll is infrastructure lifecycle, not test state

### 10. Custom Render with Real Providers (Sociable Tests)
- `createWrapper` with real Redux store, QueryClient, MemoryRouter
- `cut()` accepts UI element + options (preloadedState, route)
- Real vs fake decision table

### 11. Red Flags Table
- Mocking child components, shallow rendering, internal state access
- Module-level fetch mocking, beforeEach for component setup
- Snapshot testing as primary strategy

### 12. Common Mistakes Table
- fireEvent over userEvent, testing single-use hooks directly
- Mocking Redux store, waitFor with side-effects
- One giant integration test per page

## Research Sources

- [Next.js Official Testing Guide](https://nextjs.org/docs/app/guides/testing)
- [Testing Without Mocks - James Shore](http://www.jamesshore.com/v2/blog/2018/testing-without-mocks)
- [Sociable Tests - Dylan Watson](https://dev.to/dylanwatsonsoftware/socialise-your-unit-tests-5da0)
- [Mocks Aren't Stubs - Martin Fowler](https://martinfowler.com/articles/mocksArentStubs.html)
- [Testing Async RSCs - Aurora Scharff](https://aurorascharff.no/posts/running-tests-with-rtl-and-vitest-on-internationalized-react-server-components-in-nextjs-app-router/)
- [Kent C. Dodds - Write Tests](https://kentcdodds.com/blog/write-tests)
- [RTL Server Components Support Discussion](https://github.com/testing-library/react-testing-library/issues/1209)
- [Redux Testing Guide](https://redux.js.org/usage/writing-tests)

## TDD Validation Plan

### Baseline (without skill)
- Give agent a Next.js component + Server Action to test
- Expect: shallow rendering, jest.mock for child components, beforeEach shared state, mock handler assertions, snapshot tests

### With Skill
- Expect: cut() factory, full tree rendering, userEvent + DOM assertions, MSW for network, real providers, Server Actions tested as async functions

### Refactor
- Identify new rationalizations and add explicit counters
