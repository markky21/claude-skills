---
name: classical-testing
description: Use when writing or reviewing unit tests, integration tests, or deciding what to test. Use when you see beforeEach/afterEach shared state, London-school mocking of internal collaborators, tests coupled to implementation details, or missing test coverage for domain logic and mappers.
---

# Classical Testing

## Overview

Black-box testing based on Khorikov's "Unit Testing" book and the Classical/Detroit school. Core principle: **test observable behavior through the public API; isolate tests from each other, not units from their dependencies.**

**Prefer output-based testing. Push logic into pure functions. Mock only unmanaged external dependencies.**

**REQUIRED BACKGROUND:** superpowers:test-driven-development for TDD workflow.
**Vue-specific patterns:** See vue-classical-testing for component rendering contract tests.

## What to Test — Khorikov's 4 Quadrants

| | Few collaborators | Many collaborators |
|---|---|---|
| **High complexity / domain significance** | **Domain & Algorithms** — Unit test (90%+ coverage) | **Overcomplicated code** — Refactor first, then test |
| **Low complexity / domain significance** | **Trivial code** — Don't test | **Controllers / Orchestrators** — Integration test |

**Rules:**
- Domain models, value objects, mappers, pure functions → **Unit test thoroughly**
- Application services / controllers → **Integration test** (2 per use case: happy path + key edge case)
- Trivial code (simple getters, assignments, delegation) → **Skip**
- Complex code with many dependencies → **Refactor** into domain logic + thin controller

## The 4 Pillars — Evaluate Every Test

Every test must score well on ALL four. A zero in any pillar makes the test worthless:

1. **Protection against regressions** — Does it exercise meaningful, complex code?
2. **Resistance to refactoring** — Will it survive internal changes without false positives?
3. **Fast feedback** — Milliseconds (unit) or seconds (integration)?
4. **Maintainability** — Can someone read Arrange-Act-Assert top to bottom?

## Testing Styles — Preference Order

### 1. Output-based (most preferred)

Verify the return value. Pure functions, domain logic, mappers.

```typescript
it('should calculate discount for premium customer', () => {
  const discount = calculateDiscount(makeCustomer({ tier: 'premium' }), 100);
  expect(discount).toBe(15);
});
```

### 2. State-based (acceptable)

Verify state change through the public API. Domain aggregates, value objects.

```typescript
it('should add item to order', () => {
  const order = sut();
  order.addItem(makeProduct({ price: 10 }), 2);
  expect(order.totalPrice).toBe(20);
  expect(order.itemCount).toBe(1);
});
```

### 3. Communication-based (only for unmanaged out-of-process dependencies)

Verify interaction with external systems other applications observe. Email, message bus, third-party APIs.

```typescript
it('should send confirmation email after order placed', async () => {
  const emailGateway = { send: vi.fn() };
  const result = await sut({ emailGateway }).submitOrder(makeOrder());
  expect(emailGateway.send).toHaveBeenCalledWith(
    expect.objectContaining({ type: 'order-confirmation' })
  );
});
```

**Never mock managed dependencies** (your own database, internal repos). Use real implementations or in-memory fakes.

## SuT / CuT Factory Pattern

**Every test file uses a factory function instead of beforeEach/afterEach.** The factory encapsulates all setup so each test is self-contained.

### Unit Test — `sut()` returns the result or configured object

```typescript
const defaultProps = {
  customerId: 'customer-1',
  items: [makeOrderItem()],
};

const sut = (overrides: Partial<typeof defaultProps> = {}) => {
  const props = { ...defaultProps, ...overrides };
  const order = new Order('order-1', props.customerId);
  props.items.forEach(item =>
    order.addItem(makeProduct({ id: item.productId, price: item.unitPrice }), item.quantity)
  );
  return order;
};

it('should calculate total price from items', () => {
  const order = sut({ items: [makeOrderItem({ unitPrice: 10, quantity: 3 })] });
  expect(order.totalPrice).toBe(30);
});
```

### Integration Test — `sut()` returns configured system with overridable dependencies

```typescript
interface SutOverrides {
  orderRepo?: OrderRepository;
  emailGateway?: Partial<EmailGateway>;
  eventBus?: Partial<EventBus>;
  seedOrders?: Order[];
}

const sut = async (overrides: SutOverrides = {}) => {
  const orderRepo = overrides.orderRepo ?? new InMemoryOrderRepo();
  if (overrides.seedOrders) {
    for (const order of overrides.seedOrders) await orderRepo.save(order);
  }
  const emailGateway = { send: vi.fn(), ...overrides.emailGateway } as EmailGateway;
  const eventBus = { publish: vi.fn(), ...overrides.eventBus } as EventBus;
  const controller = new OrderController(orderRepo, emailGateway, eventBus);
  return { controller, orderRepo, emailGateway, eventBus };
};

it('should submit order and send confirmation', async () => {
  const order = makeOrder({ id: 'order-1', withItems: true });
  const { controller, emailGateway } = await sut({ seedOrders: [order] });

  await controller.submitOrder('order-1');

  expect(emailGateway.send).toHaveBeenCalledWith(
    expect.objectContaining({ type: 'order-confirmation' })
  );
});
```

**Key rules:**
- `sut()` accepts overrides for test-relevant values only; sensible defaults for everything else
- Mocks and fakes live inside `sut()` — no shared `let` variables
- Each test calls `sut()` directly — fully self-contained, order-independent
- Helper factories for test data: `makeOrder()`, `makeProduct()`, `makeOrderItem()`

### Mapper Test — `sut()` returns mapped result

```typescript
const sut = (overrides: Partial<OrderEntity> = {}) => {
  const entity: OrderEntity = {
    id: 'order-1',
    customer_id: 'cust-1',
    status: 'draft',
    items: [makeOrderItemEntity()],
    ...overrides,
  };
  return OrderMapper.toDomain(entity);
};

it('should map entity status to domain enum', () => {
  const order = sut({ status: 'shipped' });
  expect(order.status).toBe(OrderStatus.SHIPPED);
});
```

**Mappers must be fully unit tested** — both directions plus round-trip.

## Test Naming

**Describe behavior in plain English, not method names.**

```typescript
// ✅ GOOD — behavior-focused
it('should reject order submission when cart is empty', ...)
it('should merge quantities when adding the same product twice', ...)
it('Delivery_with_a_past_date_is_invalid', ...)

// ❌ BAD — method-focused
it('addItem_DuplicateProduct_IncreasesQuantity', ...)
it('submit_EmptyOrder_ThrowsError', ...)
it('should call findById', ...)
```

**Group by behavior/scenario, not by method:**

```typescript
// ✅ GOOD
describe('Order submission', () => { ... })
describe('Adding items to a draft order', () => { ... })

// ❌ BAD
describe('addItem', () => { ... })
describe('submit', () => { ... })
```

## Unit vs Integration Test Boundaries

### Unit Tests — Domain Layer

Cover these with high confidence (90%+):
- **Domain models & aggregates** — business rules, invariants, state transitions
- **Value objects** — equality, validation, immutability
- **Mappers** — DTO↔Entity, Entity↔Persistence, both directions + round-trip
- **Pure functions & utilities** — calculations, transformations, validators

Characteristics: no I/O, no async, no infrastructure. Millisecond execution.

### Integration Tests — Application Layer

Cover with 2 tests per use case (happy path + key edge case):
- **Application services / controllers** — orchestration of domain + infrastructure
- Use **real managed dependencies** (in-memory repo, test database) where feasible
- **Mock only unmanaged** out-of-process dependencies (email gateway, external message bus, third-party APIs)

Characteristics: may involve I/O, async, real infrastructure. Seconds execution.

### Mock Boundary — The Critical Distinction

| Dependency type | Example | Mock it? |
|---|---|---|
| **In-process, domain** | Other domain classes, value objects | **Never** — use real objects |
| **Managed out-of-process** | Your own database, file storage | **No** — use in-memory fakes |
| **Unmanaged out-of-process** | SMTP, external message bus, third-party API | **Yes** — these are observable by other systems |

## Red Flags — STOP and Reconsider

| Pattern | Problem |
|---------|---------|
| `beforeEach(() => { sut = ... })` | Shared mutable state between tests |
| `let repo; let service; beforeEach(...)` | Hidden setup — put inside `sut()` factory |
| `jest.mock('./internal-module')` | Mocking in-process collaborators (London school) |
| `expect(repo.save).toHaveBeenCalledWith(...)` | Verifying managed dependency interaction |
| `expect(repo.findById).toHaveBeenCalled()` | Testing internal wiring, not behavior |
| `(service as any).privateMethod()` | Accessing implementation details |
| `[MethodName]_[Scenario]_[Result]` names | Couples test names to implementation |
| Testing getters/setters/trivial code | No domain significance — skip |
| One giant integration test per feature | Split: 2 focused tests per use case |
| `spy(internalService, 'calculate')` | Verifying intra-system communication |
| Testing execution order of side effects | Implementation detail — order may change |
| `(mock as ReturnType<typeof vi.fn>)` recasting | Symptom of shared mutable mock state |

**All of these mean: restructure the test. Use `sut()` factory. Test observable behavior.**

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `beforeEach`/`afterEach` for test setup | `sut()` factory function with overrides |
| Mocking your own repo/database | In-memory fake implementing the same interface |
| One assertion per mock call (London style) | One integration test verifying the whole use case outcome |
| Organizing tests by method name | Organize by behavior/scenario |
| Testing internal method call sequence | Test the end result — what did the user/client observe? |
| Separate "should call X" tests for each dependency | One test: seed data → act → verify outcome + external effects |
| Skipping mapper tests | Mappers are domain-adjacent — test both directions + round-trip |
| 100% coverage as a goal | 90%+ on domain layer. Skip trivial code. Coverage is a negative indicator only. |
