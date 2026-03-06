# Next.js Classical Testing — Baseline Results

**Date:** 2026-03-06

## Baseline (WITHOUT skill)

### ProductCard Component
- **Mocked child component:** `vi.mock('@/components/PriceDisplay')` with custom implementation — London school
- **`beforeEach` shared state:** `let onAddToCart; beforeEach(() => { vi.clearAllMocks(); onAddToCart = vi.fn() })`
- **Mock callback assertion:** `expect(onAddToCart).toHaveBeenCalledWith(mockProduct)` — tests click→callback wiring
- **Verified PriceDisplay was called:** `expect(PriceDisplay).toHaveBeenCalledWith(expect.objectContaining({ price: 49.99 }))` — internal wiring
- **`getByTestId` usage:** `screen.getByTestId("price-display")` on mocked component
- **No `cut()` factory:** Top-level `mockProduct` constant, separate `let` variables

### useProducts Hook
- **`vi.stubGlobal('fetch')`:** Module-level fetch mocking instead of MSW network boundary
- **`beforeEach`/`afterEach`:** `vi.stubGlobal` in beforeEach, `vi.restoreAllMocks` in afterEach
- **`expect(fetch).toHaveBeenCalledWith("/api/products")`:** Verifying internal wiring
- **No `sut()` factory:** Direct `renderHook(() => useProducts())` calls

### createProduct Server Action
- **Mocked managed dependency:** `vi.mock('@/lib/productRepo')` — repo is your own database layer
- **`beforeEach(() => { vi.clearAllMocks() })`:** Shared mock state
- **`expect(productRepo.save).toHaveBeenCalledWith(...)`:** Verifying managed dependency interaction
- **`expect(productRepo.save).not.toHaveBeenCalled()`:** Verifying negative wiring
- **`expect(revalidatePath).toHaveBeenCalledWith("/products")`:** Acceptable (unmanaged Next.js primitive)
- **No `sut()` factory:** Inline `createFormData()` helper but no SuT pattern

### GET /api/products Route Handler
- **Mocked managed dependency:** `vi.mock('@/lib/productRepo')` — same as Server Action
- **`beforeEach(() => { vi.clearAllMocks() })`:** Shared mock state
- **`expect(productRepo.findAll).toHaveBeenCalledTimes(1)`:** Testing internal wiring
- **`expect(productRepo.findByCategory).not.toHaveBeenCalled()`:** Verifying negative execution path
- **No `sut()` factory:** Inline `createRequest()` helper

### Summary of Violations

| Violation | Occurrences |
|---|---|
| `beforeEach` shared state | 4/4 files |
| No `cut()`/`sut()` factory | 4/4 files |
| Mocked managed dependencies (productRepo) | 2/4 files |
| Mock callback/wiring assertions | 3/4 files |
| Module-level fetch mocking (not MSW) | 1/4 files |
| Mocked child component | 1/4 files |
| `getByTestId` on mocked component | 1/4 files |
