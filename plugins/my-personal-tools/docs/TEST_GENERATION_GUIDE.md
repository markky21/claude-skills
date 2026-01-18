# Test Generation Guide for Validator Developers

When your validator is integrated with fix-loop's test generation system, you can provide context that helps the test generator write better tests.

## Overview

After your validator fixes code, the test-generator agent runs. It uses:
1. Git diff (what changed)
2. **Validator context** (why it changed) ← You control this
3. Project patterns (how to test)

This guide explains how to provide clear context so tests are meaningful.

## Providing Validator Context

When your validator output includes fixes, add context that explains:
- **What design pattern or logic was introduced** - "Extracted domain validation logic"
- **What public APIs were created** - "New UserRegistrationService class with registerUser() method"
- **What business logic matters** - "Email validation must run before user creation"

### Example: DDD/OOP Validator

Your validator might fix anemic models by extracting business logic:

```
🔴 CRITICAL: Anemic domain model - User has no behavior
   📍 src/domain/User.ts:8
   💡 Add validateEmail method to User entity

CONTEXT FOR TEST GENERATION:
- Pattern: DDD domain model enrichment
- New public methods: validateEmail()
- Business logic: Email must contain @ symbol
- Test focus: Validation rules and error cases
```

The test generator will then create:

```typescript
describe('User', () => {
  describe('validateEmail', () => {
    it('should accept valid email formats', () => { ... });
    it('should reject invalid email formats', () => { ... });
  });
});
```

### Example: DRY Validator

Your validator might extract duplicated constants:

```
🟠 HIGH: Magic string duplicated 3 times
   📍 src/config.ts, src/utils.ts, src/api.ts (all use "USER_ROLE")
   💡 Create USER_ROLE constant in constants.ts, import everywhere

CONTEXT FOR TEST GENERATION:
- Pattern: DRY constant extraction
- New constant: USER_ROLE in src/constants.ts
- Usage locations: config.ts, utils.ts, api.ts
- Test focus: Constant is used correctly, has expected value
```

Test generator will verify the constant works correctly.

## Output Format for Validator Context

Include context in your validator output like this:

```
🔴 CRITICAL: [Issue]
   📍 file:line
   💡 [Recommendation]

   TEST_CONTEXT:
   - Pattern: [What you introduced: "DDD service", "Extracted constant", "Mapper", etc.]
   - Creates: [What new code was created: classes, methods, functions]
   - Public API: [What's publicly callable: method names, class names]
   - Business Logic: [What matters to test: validation rules, behavior]
   - Test Focus: [What tests should verify: happy paths, errors, side effects]
```

### Full Example

```
🔴 CRITICAL: User registration has no business logic validation
   📍 src/services/UserService.ts:12
   💡 Create UserRegistrationService with validateEmail and checkDuplicate methods

   TEST_CONTEXT:
   - Pattern: DDD service extraction
   - Creates: UserRegistrationService class
   - Public API: registerUser(id, email), validateEmail(email), checkDuplicate(email)
   - Business Logic:
     * Email validation: must contain @
     * Duplicate check: query repository before creating
     * Creation: only after both validations pass
   - Test Focus: Happy path registration, validation errors, duplicate detection
```

## When Context is Optional

You don't always need context. The test generator will attempt to understand code from:
- Git diff alone
- Method/class names
- Existing project test patterns

Context is most helpful for:
- Complex business logic that intent isn't obvious
- Multiple related changes that form one feature
- Non-standard patterns that need explanation
- Error cases that should be tested

## Best Practices

1. **Be specific** - "DDD service" is good, but "Service with domain validation logic" is better
2. **List all public methods** - So tests know what to call
3. **Highlight business rules** - "Email must contain @, be <256 chars"
4. **Mention dependencies** - "Uses UserRepository for duplicate checking"
5. **Keep it concise** - 3-5 bullets per fix, not paragraphs

## Example: Complete Validator Output

```
🔴 CRITICAL: User entity is anemic (data only, no behavior)
   📍 src/domain/User.ts:1
   💡 Move email validation logic into User entity as private validateEmail() method

   TEST_CONTEXT:
   - Pattern: DDD - move validation behavior to entity
   - Creates: User class with private validateEmail(email) method
   - Public API: constructor(id, email) - should validate on creation
   - Business Logic: Email format validation (must contain @)
   - Test Focus: Constructor validation, error on invalid email, success on valid email

🟠 HIGH: User service has no duplicate checking
   📍 src/services/UserService.ts:8
   💡 Extract duplicate checking into UserRegistrationService.registerUser()

   TEST_CONTEXT:
   - Pattern: Service extraction with dependency injection
   - Creates: UserRegistrationService(userRepository) class with registerUser method
   - Public API: registerUser(id, email) -> Promise<User>
   - Business Logic: Check repository for existing user, throw error if found, create if not
   - Dependencies: Depends on UserRepository interface (mock in tests)
   - Test Focus: Happy path creation, error on duplicate, repository interaction
```

## Integration with Test Generator

The test generator will:
1. Parse your context from the fixing findings
2. Understand the design pattern you introduced
3. Generate tests aligned with your stated business logic
4. Create tests for stated public API
5. Focus on areas you marked as important

The more specific your context, the better the tests will be.

## Questions?

If test generator creates tests that don't match your intent:
1. Review your context - was it clear?
2. Make context more specific or complete
3. You can also manually adjust generated tests if needed

Remember: generated tests are a starting point. They can always be refined manually.
