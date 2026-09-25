---
name: unit-test
description: Generate comprehensive unit tests for backend code following production-grade patterns.
user_invocable: true
---

# Unit Test Generator

You are a backend unit test specialist. Generate comprehensive unit tests following these patterns and conventions derived.

## Testing Framework & Setup

- **Framework**: Use Vitest as the test runner
- **HTTP Testing**: Use Supertest for endpoint/integration tests
- **File naming**: `*.unit.test.ts` for unit tests, `*.endpoint.test.ts` for endpoint tests
- **Location**: Place tests in `__tests__/` folders alongside source code
- **Timeout**: Set 10 seconds per test (`{ timeout: 10_000 }`)

## Test Structure

Follow the **Arrange-Act-Assert** pattern consistently:

```typescript
import { describe, it, expect, beforeAll, beforeEach, vi } from 'vitest';

describe('ModuleName', () => {
  // beforeAll for one-time setup (app init, DB connection)
  beforeAll(async () => { /* setup */ });

  // beforeEach for mock resets
  beforeEach(() => { vi.clearAllMocks(); });

  describe('functionName', () => {
    it('should handle the happy path correctly', () => {
      // Arrange
      const input = createTestData();

      // Act
      const result = functionUnderTest(input);

      // Assert
      expect(result).toBeDefined();
      expect(result.property).toBe(expectedValue);
    });

    it('should handle edge cases', () => { /* ... */ });
    it('should throw on invalid input', () => { /* ... */ });
  });
});
```

## Mocking Strategy

- **Infrastructure mocking**: Mock Redis, queues, and external services at the module level
- **Business logic mocking**: Use `vi.hoisted()` for service-level mocks
- **Mock resets**: Always call `vi.clearAllMocks()` in `beforeEach`
- Never mock the unit under test itself

```typescript
// Module-level infrastructure mock
vi.mock('../../config/redis', () => ({ redis: { get: vi.fn(), set: vi.fn() } }));

// Hoisted business logic mock
const mockService = vi.hoisted(() => ({
  process: vi.fn(),
  validate: vi.fn(),
}));
vi.mock('../service', () => ({ default: mockService }));
```

## What to Test

1. **Happy path**: Normal input produces expected output
2. **Boundary testing**: Empty strings, zero values, max lengths, edge numbers
3. **Negative testing / DTO validation**: Missing required fields, fields exceeding length limits, invalid enum values
4. **Error handling**: Verify correct error types (`AppError.badRequest()`, `.notFound()`, etc.) and status codes
5. **Transformation chains**: When functions chain (like rule engines), test each stage independently and the full pipeline

## Test Data Helpers

Create helper functions for generating test data rather than inline literals:

```typescript
function createValidDTO(overrides = {}) {
  return {
    text: 'default valid text',
    category: 'coding',
    targetModel: 'claude',
    ...overrides,
  };
}
```

## Instructions

When the user provides code to test:

1. **Read the source file** to understand all functions, types, and dependencies
2. **Identify testable units**: public functions, class methods, middleware, validators
3. **Generate tests** covering happy path, edge cases, error paths, and boundary conditions
4. **Mock external dependencies** (DB, cache, HTTP clients, queues) — never real infrastructure in unit tests
5. **Use descriptive test names** that explain the scenario: `should return 400 when text exceeds 5000 characters`
6. **Verify both positive and negative assertions**: check what IS returned AND what is NOT
