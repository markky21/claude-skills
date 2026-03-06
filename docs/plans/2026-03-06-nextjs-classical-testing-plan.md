# Next.js Classical Testing Skill — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a `nextjs-classical-testing` skill that extends `classical-testing` with React/Next.js-specific patterns for component rendering, interactions, hooks, Server Actions, Route Handlers, async RSC, MSW, and sociable tests.

**Architecture:** Single SKILL.md file following the same structure as `vue-classical-testing`. References `classical-testing` as required background. Uses Vitest examples with Jest notes where they differ.

**Tech Stack:** React Testing Library, Vitest/Jest, MSW, userEvent, Next.js App Router

---

### Task 1: RED — Run Baseline Test (Without Skill)

**Purpose:** Document what an agent does WITHOUT the skill to identify rationalizations and bad patterns.

**Step 1: Create a pressure scenario prompt**

Write the following prompt and dispatch it to a subagent WITHOUT the `nextjs-classical-testing` skill loaded:

```
You are testing a Next.js application. Write tests for:

1. A `ProductCard` component that receives `product` props and renders name, price, and an "Add to Cart" button. When clicked, it calls an `onAddToCart` callback. It uses a child `PriceDisplay` component.

2. A `useProducts` hook that fetches products from `/api/products` and returns `{ products, isLoading, error }`.

3. A Server Action `createProduct(formData: FormData)` that validates input, saves to database via `productRepo.save()`, and calls `revalidatePath('/products')`.

4. A Route Handler `GET /api/products` that returns products as JSON, with optional `?category=` filter.

Use Vitest and React Testing Library. Create complete test files.
```

**Step 2: Run the baseline scenario**

Dispatch the prompt above to a subagent (type: general-purpose). Do NOT mention classical testing or the skill.

**Step 3: Document baseline behavior**

Record verbatim:
- Did the agent use `beforeEach`/shared `let` variables?
- Did it mock child components (`jest.mock('./PriceDisplay')`)?
- Did it test `onAddToCart` callback via `expect(mockFn).toHaveBeenCalled()`?
- Did it use `vi.mock('fetch')` or module-level mocking?
- Did it use `shallow` or mock internal state?
- Did it test the hook with `vi.mock` for fetch instead of MSW?
- How did it structure Server Action tests? Did it mock `productRepo`?
- Did it use snapshot tests?

**Step 4: Save baseline results**

Save findings to `docs/plans/2026-03-06-nextjs-classical-testing-baseline.md`

**Step 5: Commit**

```bash
git add docs/plans/2026-03-06-nextjs-classical-testing-baseline.md
git commit -m "test(skill): baseline results for nextjs-classical-testing"
```

---

### Task 2: GREEN — Write the Skill

**Files:**
- Create: `plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p plugins/my-personal-tools/skills/nextjs-classical-testing
```

**Step 2: Write the SKILL.md**

Create `plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md` with this content (adapt based on baseline findings — add explicit counters for rationalizations observed):

```markdown
---
name: nextjs-classical-testing
description: Use when writing or reviewing React/Next.js component tests, hook tests, Server Action tests, or Route Handler tests. Use when you see shallow rendering, excessive jest.mock(), implementation-detail assertions, or London-school mocking of internal collaborators in a Next.js codebase.
---

# Next.js Classical Testing

## Overview

Classical (blackbox) testing for React and Next.js applications using React Testing Library. Test the rendering contract and observable behavior — props in, DOM out, user interactions verified by DOM changes.

**REQUIRED BACKGROUND:** classical-testing for general unit/integration testing principles (Khorikov's 4 quadrants, SuT/CuT pattern, mock boundaries). This skill extends those principles to React/Next.js.

**Sources:**
- "The more your tests resemble how your software is used, the more confidence they can give you" — React Testing Library
- "Write tests. Not too many. Mostly integration." — Kent C. Dodds

## What to Test — Next.js Feature Map

| Feature | Test approach | Tool |
|---|---|---|
| Client Components | Rendering contract + interactions | Vitest/Jest + RTL |
| Hooks (shared, 3+ consumers) | Output-based via `renderHook` | Vitest/Jest + RTL |
| Hooks (single component) | Test through the component | Vitest/Jest + RTL |
| Server Actions | Plain async function test | Vitest/Jest |
| Route Handlers | Direct import, NextRequest | Vitest/Jest |
| Sync Server Components | Rendering contract with RTL | Vitest/Jest + RTL |
| Async Server Components | Workarounds or E2E | See Async RSC section |
| Data fetching | MSW at network boundary | MSW + Vitest/Jest |
| Full user flows | E2E | Playwright/Cypress |

## Component Rendering Contracts — `cut()` Factory

Use a `cut` factory function that encapsulates **all** setup so each test is fully self-contained. No `beforeEach`, `beforeAll`, or shared mutable state.

```typescript
import { render, screen } from '@testing-library/react'

const defaultProps = {
  items: [createItem()],
  title: 'My List',
}

const cut = (propsOverrides: Partial<typeof defaultProps> = {}) => {
  return render(<ItemList {...defaultProps} {...propsOverrides} />)
}

describe('ItemList', () => {
  it('should render item labels from props', () => {
    cut({ items: [createItem({ label: 'Apple' }), createItem({ label: 'Banana' })] })
    expect(screen.getByText('Apple')).toBeInTheDocument()
    expect(screen.getByText('Banana')).toBeInTheDocument()
  })

  it('should render empty state when no items', () => {
    cut({ items: [] })
    expect(screen.getByText('No items found')).toBeInTheDocument()
  })
})
```

**RTL query priority** (test like a user):
1. `getByRole` — most accessible, most resilient
2. `getByLabelText` — form elements
3. `getByText` — visible content
4. `getByTestId` — last resort

**No `shallowMount` equivalent.** RTL always renders the full tree — child components are managed dependencies (never mock them).

## User Interaction Testing

Classical-aligned because we assert on **DOM output changes**, not handler calls:

```typescript
import userEvent from '@testing-library/user-event'

const cut = (propsOverrides = {}) => {
  const user = userEvent.setup()
  render(<Counter {...defaultProps} {...propsOverrides} />)
  return { user }
}

describe('Counter', () => {
  it('should increment displayed count on button click', async () => {
    const { user } = cut({ initialCount: 0 })
    await user.click(screen.getByRole('button', { name: /increment/i }))
    expect(screen.getByText('1')).toBeInTheDocument()
  })
})
```

**Blackbox boundary:**

| Blackbox (assert DOM) | Implementation detail |
|---|---|
| `screen.getByText('1')` after click | `expect(setState).toHaveBeenCalledWith(1)` |
| Button becomes disabled | `expect(component.state.isValid).toBe(false)` |
| Error message appears | `expect(onError).toHaveBeenCalled()` |
| New item appears in list | `expect(setItems).toHaveBeenCalledWith(...)` |

**`userEvent` over `fireEvent`:** `userEvent` simulates real browser behavior (focus, keydown, keyup, click). Classical testing prefers the more realistic simulation.

## Hooks Testing

Output-based testing with `sut()` pattern for shared hooks:

```typescript
import { renderHook, act } from '@testing-library/react'

const sut = (overrides: Partial<Parameters<typeof useCart>[0]> = {}) => {
  const props = { initialItems: [], ...overrides }
  return renderHook(() => useCart(props))
}

describe('useCart', () => {
  it('should calculate total from items', () => {
    const { result } = sut({
      initialItems: [createItem({ price: 10 }), createItem({ price: 20 })]
    })
    expect(result.current.total).toBe(30)
  })

  it('should add item and update count', () => {
    const { result } = sut()
    act(() => result.current.addItem(createItem({ price: 15 })))
    expect(result.current.itemCount).toBe(1)
  })
})
```

**When to test hook directly vs through component:**

| Test hook directly | Test through component |
|---|---|
| Shared hook used by 3+ components | Hook used by single component |
| Complex derived state / calculations | Simple useState/useEffect wiring |
| Hook is the public API (library) | Hook is implementation detail |

## Server Actions Testing

Server Actions are controllers in Khorikov's model — test as plain async functions:

```typescript
vi.mock('next/headers', () => ({
  cookies: vi.fn(() => ({ get: vi.fn(), set: vi.fn() })),
}))
vi.mock('next/navigation', () => ({
  redirect: vi.fn(),
}))
vi.mock('next/cache', () => ({
  revalidatePath: vi.fn(),
  revalidateTag: vi.fn(),
}))

const sut = (overrides: Record<string, string> = {}) => {
  const formData = new FormData()
  formData.set('name', overrides.name ?? 'Widget')
  formData.set('price', overrides.price ?? '10')
  return createProduct(formData)
}

describe('createProduct', () => {
  it('should return success for valid input', async () => {
    const result = await sut()
    expect(result).toEqual({ success: true })
  })

  it('should return validation errors for empty name', async () => {
    const result = await sut({ name: '' })
    expect(result.errors.name).toBe('Name is required')
  })
})
```

**Mock boundary for Server Actions:**

| Mock it | Don't mock it |
|---|---|
| `next/headers`, `next/navigation`, `next/cache` | Validation logic, domain functions |
| External APIs (payment, email) | Your own database (use in-memory fake) |

## Route Handler Testing

Direct import and call with constructed `NextRequest`:

```typescript
import { GET, POST } from './route'
import { NextRequest } from 'next/server'

const sut = {
  get: (params: Record<string, string> = {}) => {
    const url = new URL('http://localhost/api/products')
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v))
    return GET(new NextRequest(url))
  },
  post: (body: Record<string, unknown> = {}) => {
    return POST(new NextRequest('http://localhost/api/products', {
      method: 'POST',
      body: JSON.stringify({ name: 'Widget', price: 10, ...body }),
    }))
  },
}

describe('Products API', () => {
  it('should return products as JSON', async () => {
    const response = await sut.get()
    expect(response.status).toBe(200)
    const data = await response.json()
    expect(data.products).toEqual(expect.arrayContaining([
      expect.objectContaining({ name: expect.any(String) })
    ]))
  })

  it('should reject invalid product body', async () => {
    const response = await sut.post({ price: -1 })
    expect(response.status).toBe(400)
  })
})
```

## Async Server Components

Vitest/Jest can't natively render async Server Components. Three workarounds in preference order:

```typescript
// 1. Direct await (leaf components, simplest)
it('should render user profile', async () => {
  const Result = await UserProfile({ userId: '123' })
  render(Result)
  expect(screen.getByText('John Doe')).toBeInTheDocument()
})

// 2. Suspense wrapper (React 19+, nested async)
const cutAsync = (props = {}) => {
  render(
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile userId="123" {...props} />
    </Suspense>
  )
}

it('should render user profile', async () => {
  cutAsync()
  expect(await screen.findByText('John Doe')).toBeInTheDocument()
})

// 3. Custom renderAsync (deeply nested async trees)
async function renderAsync(node: JSX.Element) {
  if (typeof node.type === 'function' && node.type.constructor.name === 'AsyncFunction') {
    const resolved = await node.type({ ...node.props })
    return renderAsync(resolved)
  }
  return render(node)
}
```

| Workaround | Use when |
|---|---|
| Direct await | Leaf component, no nested async children |
| Suspense wrapper | React 19+, component uses Suspense naturally |
| Custom renderAsync | Deeply nested async tree |
| **Defer to E2E** | Middleware-dependent, auth-gated, complex chains |

**Note:** Evolving area. Prefer native support when tooling catches up.

## MSW for Network Boundary Mocking

Mock at the HTTP boundary, not the module level — the classical approach:

```typescript
import { http, HttpResponse } from 'msw'
import { setupServer } from 'msw/node'

const handlers = [
  http.get('/api/products', () =>
    HttpResponse.json([createProduct(), createProduct({ name: 'Widget' })])
  ),
]

const server = setupServer(...handlers)
beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

it('should show error state on API failure', async () => {
  server.use(
    http.get('/api/products', () => HttpResponse.json(null, { status: 500 }))
  )
  cut()
  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument()
})
```

**Why MSW over `vi.mock('fetch')`:**

| MSW (classical) | Module mocking (London school) |
|---|---|
| Mocks at network boundary | Mocks at import level |
| App code runs as production | Bypasses HTTP client |
| Works with any HTTP client | Coupled to specific client |

**Note:** MSW `beforeAll`/`afterAll` is infrastructure lifecycle, not test state. The `cut()` factory pattern still applies for component setup.

## Custom Render with Real Providers (Sociable Tests)

Classical testing = real collaborators. Use real providers, not mocked ones:

```typescript
interface CutOptions extends Omit<RenderOptions, 'wrapper'> {
  preloadedState?: Partial<RootState>
  store?: AppStore
  route?: string
}

const createWrapper = (options: CutOptions = {}) => {
  const {
    preloadedState = {},
    store = configureStore({ reducer: rootReducer, preloadedState }),
    route = '/',
  } = options
  window.history.pushState({}, '', route)
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
      <Provider store={store}>
        <MemoryRouter initialEntries={[route]}>
          {children}
        </MemoryRouter>
      </Provider>
    </QueryClientProvider>
  )
}
```

**Real vs fake:**

| Real (sociable) | Fake/Mock (boundary) |
|---|---|
| Redux/Zustand store | Network requests (MSW) |
| React Query client | Browser APIs not in jsdom |
| Router (MemoryRouter) | `next/navigation` in unit tests |
| Context providers, child components | External auth providers |

## Red Flags — STOP and Reconsider

| Pattern | Problem |
|---|---|
| `jest.mock('./components/ChildComponent')` | Mocking managed dependencies (London school) |
| `shallow(<Component />)` or enzyme shallow | Over-isolation, hides integration bugs |
| `(component as any).state` | Accessing internal state |
| `expect(mockFn).toHaveBeenCalledWith(...)` on internals | Testing wiring, not behavior |
| `getByTestId` as first query choice | Couples to test infrastructure |
| `vi.mock('axios')` / `jest.mock('fetch')` | Module-level mocking — use MSW |
| `beforeEach(() => { ... })` for component setup | Shared mutable state — use `cut()` factory |
| `act(() => { setState(...) })` | Directly manipulating state |
| `expect(container.innerHTML).toMatchSnapshot()` | Brittle snapshots coupled to DOM structure |

## Common Mistakes

| Mistake | Fix |
|---|---|
| Mocking child components | Render full tree — children are managed dependencies |
| `fireEvent.click` over `userEvent.click` | `userEvent` simulates real browser behavior |
| Testing hook used by one component directly | Test through the component instead |
| Snapshot testing as primary strategy | Use explicit assertions on content |
| Mocking Redux store/context | Use real store with `preloadedState` |
| Separate test for each callback | One test: interact → assert DOM changed |
| `waitFor` with side-effect inside | `waitFor` for assertions only |
| Testing `className` or style changes | Visual concern — leave to visual regression |
| One giant test per page | 2 per use case: happy path + key edge case |
```

**Step 3: Commit the skill**

```bash
git add plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md
git commit -m "feat(skills): add nextjs-classical-testing skill"
```

---

### Task 3: GREEN — Run Scenario WITH Skill

**Step 1: Run same pressure scenario with skill loaded**

Dispatch the same prompt from Task 1 to a subagent, but this time ensure the `nextjs-classical-testing` skill is available (include it in the system prompt or reference it).

**Step 2: Compare results**

Verify the agent now:
- Uses `cut()` factory instead of `beforeEach`
- Renders full component tree (no mocking of `PriceDisplay`)
- Asserts DOM changes after interactions (not `onAddToCart` mock calls)
- Uses MSW for the `useProducts` hook test (not `vi.mock('fetch')`)
- Tests Server Action as plain async function with `sut()` factory
- Tests Route Handler with direct import + `NextRequest` construction
- No snapshot tests

**Step 3: Document GREEN results**

Append findings to `docs/plans/2026-03-06-nextjs-classical-testing-baseline.md` under a "## With Skill" section.

**Step 4: Commit**

```bash
git add docs/plans/2026-03-06-nextjs-classical-testing-baseline.md
git commit -m "test(skill): GREEN results for nextjs-classical-testing"
```

---

### Task 4: REFACTOR — Close Loopholes

**Step 1: Identify new rationalizations**

From GREEN results, find any remaining violations:
- Did the agent still use `beforeEach` for MSW setup? (Acceptable — infrastructure lifecycle)
- Did it rationalize mocking child components in any edge case?
- Did it use `fireEvent` instead of `userEvent`?
- Any new patterns not covered by the skill?

**Step 2: Update SKILL.md**

For each rationalization found, add an explicit counter to the Red Flags or Common Mistakes tables.

**Step 3: Re-test if significant changes were made**

Run the pressure scenario one more time to verify the counters work.

**Step 4: Commit**

```bash
git add plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md
git commit -m "refactor(skills): close loopholes in nextjs-classical-testing"
```

---

### Task 5: Update Cross-References

**Files:**
- Modify: `plugins/my-personal-tools/skills/classical-testing/SKILL.md` (add reference to new skill)

**Step 1: Add cross-reference to parent skill**

In `classical-testing/SKILL.md`, after the existing line:
```
**Vue-specific patterns:** See vue-classical-testing for component rendering contract tests.
```

Add:
```
**React/Next.js patterns:** See nextjs-classical-testing for RTL rendering contracts, hooks, Server Actions, Route Handlers, and MSW.
```

**Step 2: Commit**

```bash
git add plugins/my-personal-tools/skills/classical-testing/SKILL.md
git commit -m "docs: cross-reference nextjs-classical-testing from classical-testing"
```

---

### Task 6: Final Verification

**Step 1: Verify skill file structure**

```bash
ls -la plugins/my-personal-tools/skills/nextjs-classical-testing/
# Expected: SKILL.md only
```

**Step 2: Verify YAML frontmatter is valid**

```bash
head -4 plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md
# Expected: ---, name: nextjs-classical-testing, description: Use when..., ---
```

**Step 3: Verify description length**

```bash
head -4 plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md | wc -c
# Expected: under 1024 characters total frontmatter
```

**Step 4: Verify word count**

```bash
wc -w plugins/my-personal-tools/skills/nextjs-classical-testing/SKILL.md
# Target: under 500 words (per writing-skills guidance for non-frequently-loaded skills)
```

**Step 5: Final commit if any adjustments needed**

```bash
git add -A && git commit -m "chore: finalize nextjs-classical-testing skill"
```
