# Classical Testing Skill Design

**Date:** 2026-03-06
**Status:** Implemented

## Goal

Create a framework-agnostic testing skill based on Khorikov's "Unit Testing: Principles, Practices, and Patterns" and the Classical/Detroit school of testing.

## Key Principles

- **Black-box testing by default** — test observable behavior through public API
- **Classical school** — isolate tests from each other, not units from dependencies
- **Output-based > State-based > Communication-based** testing preference
- **SuT/CuT factory pattern** — replace beforeEach/afterEach with factory functions
- **Mock boundary** — only mock unmanaged out-of-process dependencies (SMTP, external message bus)
- **Domain logic fully tested** — 90%+ coverage on domain models, value objects, mappers

## Khorikov's 4 Quadrants (What to Test)

| | Few collaborators | Many collaborators |
|---|---|---|
| High complexity | Unit test | Refactor first |
| Low complexity | Don't test | Integration test |

## Testing Decisions

- **Domain models, mappers, pure functions** → Unit test thoroughly
- **Application services / controllers** → Integration test (2 per use case)
- **Trivial code** → Skip
- **Managed dependencies (own DB)** → In-memory fakes, never mock
- **Unmanaged dependencies (SMTP, external bus)** → Mock

## Relationship to Existing Skills

- **Parent:** `classical-testing` (new, general principles)
- **Child:** `vue-classical-testing` (existing, Vue-specific rendering contracts)
- Cross-reference added from child to parent

## TDD Validation Results

### Baseline (without skill)
- Domain/mapper tests: good use of helpers, minor method-name grouping
- Controller tests: beforeEach + shared let variables, mocked repo, verified internal wiring (findById called), tested execution order, London-school pattern

### With Skill
- All beforeEach eliminated → sut() factory
- InMemoryOrderRepo replaces mocked repo
- Internal wiring assertions removed
- Execution order test removed
- Tests grouped by behavior not method
- Controller tests reduced from 13 to 7 (focused on use cases)
