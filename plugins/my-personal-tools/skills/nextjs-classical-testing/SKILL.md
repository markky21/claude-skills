---
name: nextjs-classical-testing
description: Use when writing or reviewing React/Next.js component tests, hook tests, Server Action tests, or Route Handler tests. Use when you see shallow rendering, excessive jest.mock(), implementation-detail assertions, or London-school mocking of internal collaborators in a Next.js codebase.
---

# Next.js Classical Testing

## Overview

Classical (blackbox) testing for React and Next.js using React Testing Library. Test the rendering contract and observable behavior — props in, DOM out, user interactions verified by DOM changes.

**REQUIRED BACKGROUND:** classical-testing for general unit/integration testing principles (Khorikov's 4 quadrants, SuT/CuT pattern, mock boundaries). This skill extends those principles to React/Next.js.

**Sources:**
- "The more your tests resemble how your software is used, the more confidence they can give you" — RTL
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

`cut` factory encapsulates **all** setup. No `beforeEach`, `beforeAll`, or shared mutable state.

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
4. `getByTestId` — last resort only

**Child components are managed dependencies — never mock them.** Render the full tree. No `vi.mock('./ChildComponent')`.

## User Interaction Testing

Assert on **DOM output changes**, not handler calls:

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
| Error message appears in DOM | `expect(onError).toHaveBeenCalled()` |
| New item appears in list | `expect(setItems).toHaveBeenCalledWith(...)` |

**`userEvent` over `fireEvent`:** `userEvent` simulates real browser behavior (focus, keydown, keyup, click). Classical testing prefers the more realistic simulation.

## Hooks Testing

Output-based with `sut()` for shared hooks:

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

| Test hook directly | Test through component |
|---|---|
| Shared hook used by 3+ components | Hook used by single component |
| Complex derived state / calculations | Simple useState/useEffect wiring |
| Hook is the public API (library) | Hook is implementation detail |

## Server Actions Testing

Server Actions are controllers — test as plain async functions. Mock only Next.js primitives (unmanaged), use in-memory fakes for your own data layer (managed):

```typescript
vi.mock('next/headers', () => ({
  cookies: vi.fn(() => ({ get: vi.fn(), set: vi.fn() })),
}))
vi.mock('next/navigation', () => ({ redirect: vi.fn() }))
vi.mock('next/cache', () => ({ revalidatePath: vi.fn(), revalidateTag: vi.fn() }))

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

**Mock boundary:**

| Mock it (unmanaged) | Don't mock it (managed) |
|---|---|
| `next/headers`, `next/navigation`, `next/cache` | Validation logic, domain functions |
| External APIs (payment, email) | Your own database / repo (use in-memory fake) |

**Do NOT `vi.mock('@/lib/productRepo')`.** Your repo is a managed dependency — use an in-memory implementation:

```typescript
// If action accepts repo as parameter (DI):
const sut = (overrides = {}) => {
  const repo = new InMemoryProductRepo()
  if (overrides.seedProducts) overrides.seedProducts.forEach(p => repo.save(p))
  const formData = new FormData()
  formData.set('name', overrides.name ?? 'Widget')
  return createProduct(formData, { productRepo: repo })
}

// If action imports repo at module level, swap via vi.mock
// but provide a REAL in-memory implementation, not a vi.fn() stub:
vi.mock('@/lib/productRepo', () => ({
  productRepo: new InMemoryProductRepo(),
}))
```

**Do NOT use `mockFn.mockClear()` between tests.** Each test should call `sut()` which creates fresh state. If you need to verify an unmanaged dependency call (like `revalidatePath`), create the mock inside `sut()` or use a fresh spy per test.

## Route Handler Testing

Direct import, construct `NextRequest` in `sut()`:

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
})
```

## Async Server Components

Vitest/Jest can't natively render async Server Components. Workarounds in preference order:

```typescript
// 1. Direct await (leaf components)
it('should render user profile', async () => {
  const Result = await UserProfile({ userId: '123' })
  render(Result)
  expect(screen.getByText('John Doe')).toBeInTheDocument()
})

// 2. Suspense wrapper (React 19+)
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
```

| Workaround | Use when |
|---|---|
| Direct await | Leaf component, no nested async children |
| Suspense wrapper | React 19+, component uses Suspense naturally |
| **Defer to E2E** | Middleware-dependent, auth-gated, complex chains |

## MSW for Network Boundary Mocking

Mock at HTTP boundary, not module level:

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

**MSW `beforeAll`/`afterAll` is infrastructure lifecycle, not test state.** The `cut()` factory still applies for component setup.

**Do NOT use `vi.stubGlobal('fetch')` or `vi.mock('axios')`.** Module-level mocking bypasses your HTTP client — MSW keeps all app code running as production.

## Custom Render with Real Providers

Classical = real collaborators. Use real providers, not mocked ones:

```typescript
const createWrapper = (options: { preloadedState?: Partial<RootState>, route?: string } = {}) => {
  const { preloadedState = {}, route = '/' } = options
  const store = configureStore({ reducer: rootReducer, preloadedState })
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={new QueryClient({ defaultOptions: { queries: { retry: false } } })}>
      <Provider store={store}>
        <MemoryRouter initialEntries={[route]}>{children}</MemoryRouter>
      </Provider>
    </QueryClientProvider>
  )
}
```

| Real (sociable) | Fake/Mock (boundary) |
|---|---|
| Redux/Zustand store | Network requests (MSW) |
| React Query client | Browser APIs not in jsdom |
| Router (MemoryRouter) | `next/navigation` in unit tests |
| Context providers, child components | External auth providers |

## Red Flags — STOP and Reconsider

| Pattern | Problem |
|---|---|
| `vi.mock('./components/ChildComponent')` | Mocking managed dependencies (London school) |
| `shallow(<Component />)` | Over-isolation, hides integration bugs |
| `expect(PriceDisplay).toHaveBeenCalledWith(...)` | Verifying child component was called — internal wiring |
| `expect(onCallback).toHaveBeenCalledWith(...)` | Testing click→callback wiring, not DOM output |
| `vi.mock('@/lib/productRepo')` | Mocking your own data layer — use in-memory fake |
| `expect(repo.save).toHaveBeenCalledWith(...)` | Verifying managed dependency interaction |
| `expect(repo.findAll).toHaveBeenCalledTimes(1)` | Testing internal wiring, not response output |
| `vi.stubGlobal('fetch')` / `vi.mock('axios')` | Module-level mocking — use MSW |
| `getByTestId` as first query choice | Couples to test infrastructure |
| `beforeEach(() => { wrapper = ... })` | Shared mutable state — use `cut()` factory |
| `let onAddToCart; beforeEach(...)` | Hidden setup — put inside `cut()` |
| `mockFn.mockClear()` between tests | Fresh `sut()` per test instead |
| `(component as any).state` | Accessing internal state |

**All of these mean: restructure the test. Use `cut()`/`sut()` factory. Test observable behavior.**

## Common Mistakes

| Mistake | Fix |
|---|---|
| Mocking child components | Render full tree — children are managed dependencies |
| `fireEvent.click` over `userEvent.click` | `userEvent` simulates real browser behavior |
| Testing hook used by one component directly | Test through the component instead |
| Snapshot testing as primary strategy | Use explicit assertions on content |
| Mocking Redux store/context | Use real store with `preloadedState` |
| Separate test for each callback prop | One test: interact → assert DOM changed |
| `waitFor` with side-effect inside | `waitFor` for assertions only |
| Testing `className` or style changes | Visual concern — leave to visual regression |
| One giant test per page | 2 per use case: happy path + key edge case |
| `vi.mock` for your own repo/service | In-memory fake implementing same interface |
