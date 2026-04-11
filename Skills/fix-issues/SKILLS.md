---
name: fix-issues
description: Diagnose and fix backend bugs and issues following systematic debugging patterns.
user_invocable: true
---

# Fix Issues

You are a senior backend debugger and issue resolver. Diagnose and fix bugs systematically following patterns derived.

## Diagnostic Process

### Step 1: Reproduce & Understand
- Read the error message, stack trace, or issue description carefully
- Identify the **entry point** — which route, controller, or service is involved
- Trace the request flow: route → middleware → controller → service → database/cache
- Check if the issue is in validation (DTO/Zod), business logic (service), data layer (Mongoose/Redis), or infrastructure (queue/cron)

### Step 2: Isolate the Root Cause
- **Validation errors (400)**: Check Zod DTO schemas — missing fields, wrong types, constraint mismatches
- **Auth errors (401/403)**: Check JWT/PASETO middleware, token expiry, RS256 key pair configuration
- **Not found (404)**: Verify route registration in `app.module.ts`, check Mongoose query filters
- **Conflict (409)**: Check unique index constraints, race conditions in concurrent writes
- **Rate limit (429)**: Check rate limiter config, Redis connectivity for distributed limiting
- **Server errors (500)**: Unhandled promise rejections, missing null checks, DB connection failures

### Step 3: Common Bug Categories

#### Database Issues
```typescript
// BAD: Missing await on async Mongoose operation
const user = UserModel.findById(id); // Returns Query, not document

// GOOD:
const user = await UserModel.findById(id);

// BAD: Not handling null from findById
const user = await UserModel.findById(id);
user.name = 'new'; // TypeError if user is null

// GOOD:
const user = await UserModel.findById(id);
if (!user) throw AppError.notFound('User not found');
```

#### Middleware Chain Issues
```typescript
// BAD: Wrong middleware order — auth runs before validation
router.post('/action', tokenGuard, validateDTO(schema), controller);

// GOOD: Validate first, then authenticate
router.post('/action', validateDTO(schema), tokenGuard, controller);

// BAD: Missing next() call in middleware
const middleware = (req, res) => { /* forgot next() */ };

// GOOD:
const middleware = (req, res, next) => { /* ... */ next(); };
```

#### Type Safety Issues
```typescript
// BAD: Using 'any' hides real type errors
const data: any = await service.process(input);

// GOOD: Proper typing reveals issues at compile time
const data: ProcessResult = await service.process(input);

// BAD: Optional chaining masking actual bugs
const value = obj?.deeply?.nested?.value ?? 'default';
// If 'nested' should always exist, this hides a real bug

// GOOD: Fail explicitly if structure is unexpected
if (!obj.deeply.nested) throw AppError.internal('Unexpected null in nested');
```

#### Redis/Cache Issues
```typescript
// BAD: Not handling Redis connection failure
const cached = await redis.get(key); // Crashes if Redis is down

// GOOD: Graceful degradation
let cached: string | null = null;
try {
  cached = await redis.get(key);
} catch (err) {
  logger.warn({ err }, 'Redis unavailable, falling back to DB');
}
```

#### Queue/Worker Issues
```typescript
// BAD: Not handling job failure
queue.add('email', { to, subject, body });

// GOOD: Configure retries and failure handling
queue.add('email', { to, subject, body }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
});
```

### Step 4: Fix & Verify

1. **Apply the minimal fix** — change only what's necessary to resolve the issue
2. **Check for similar patterns** — if a bug exists in one place, grep for the same pattern elsewhere
3. **Add/update tests** — write a test that would have caught this bug
4. **Verify error handling** — ensure the fix produces proper `AppError` types, not raw errors
5. **Check side effects** — does the fix affect other middleware, routes, or services?

## Error Pattern Reference

| Symptom | Likely Cause | Where to Look |
|---------|-------------|---------------|
| `Cannot read property of undefined` | Missing null check or await | Service layer, Mongoose queries |
| `ValidationError` | Schema mismatch | DTO Zod schemas |
| `MongoServerError: E11000` | Duplicate key violation | Unique indexes, upsert logic |
| `JsonWebTokenError` | Invalid/expired token | Auth middleware, key config |
| `ECONNREFUSED` | Service down | Redis, MongoDB, external API connections |
| `TimeoutError` | Slow query/operation | DB indexes, missing `.lean()`, large payloads |
| `UnhandledPromiseRejection` | Missing try-catch or `.catch()` | Async middleware, service calls |

## Instructions

When the user reports a bug or issue:

1. **Read the error/symptoms** — get the full stack trace, HTTP status, and request details
2. **Trace the code path** — follow the request from route to response
3. **Read all involved files** — controller, service, middleware, schema, DTO
4. **Identify root cause** — use the categories above to narrow down
5. **Apply minimal fix** — change only what's needed, explain why
6. **Check for similar bugs** — grep the codebase for the same anti-pattern
7. **Suggest a test** — describe or write a test that catches this regression
