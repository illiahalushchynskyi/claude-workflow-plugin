---
step_number: 1
name: Create Token Generation Service
status: pending
iteration: 1
created: 2026-04-08T10:00:00Z
started: null
completed: null
estimated_effort: medium
blocked: false
blocked_reason: null
tags:
  - authentication
  - core
  - backend
depends_on: []
---

# Step 1: Create Token Generation Service

## Goal

Create a reusable `TokenService` that generates JWT tokens with proper structure,
expiration, and verification. This service is the foundation for all authentication
in the system.

## Verification Criteria

- [ ] `TokenService.sign()` creates valid JWT tokens
- [ ] Token payload includes `userId` and `role` fields
- [ ] Tokens expire after 7 days (configurable via JWT_EXPIRY env var)
- [ ] Token can be verified with `TokenService.verify()`
- [ ] Verification throws error for invalid tokens
- [ ] Verification throws error for expired tokens
- [ ] Uses HS256 algorithm with `JWT_SECRET` from environment
- [ ] All TypeScript type definitions present (no `any` types)
- [ ] All unit tests pass
- [ ] No TypeScript compilation errors
- [ ] Code follows project conventions (eslint passes)

## Implementation

### Files to Modify/Create

- Create: `backend/src/services/TokenService.ts`
- Create: `backend/src/types/auth.ts`
- Modify: `backend/src/server.ts` (optional: add TokenService to DI)
- Create: `backend/src/services/__tests__/TokenService.test.ts`

### Implementation Notes

**Implementer**: [Will be filled in during execution]

**Timestamp**: [Will be filled in during execution]

[Implementation notes will be added as work progresses]

## Verification

### Verification Notes

**Verifier**: [Will be filled in during execution]

**Timestamp**: [Will be filled in during execution]

**Result**: [PENDING]

[Verification notes will be added after testing]

---

**History**: 
- Iteration 1: [To be updated during execution]

---

## Implementation Guidance (For Reference)

This section provides context to help implementers understand what's expected:

### Token Structure

```typescript
interface TokenPayload {
  userId: number;
  role: 'user' | 'admin';
  iat?: number;     // Issued at (auto-set by JWT library)
  exp?: number;     // Expiration (auto-set by JWT library)
}
```

### Expected API

```typescript
class TokenService {
  /**
   * Sign and return a new JWT token
   */
  static sign(userId: number, role: string): string;

  /**
   * Verify and decode a JWT token
   * @throws Error if token invalid or expired
   */
  static verify(token: string): TokenPayload;

  /**
   * Decode token without verification (for inspection)
   */
  static decode(token: string): TokenPayload;
}
```

### Environment Variables Expected

```
JWT_SECRET=your-secret-key-here
JWT_EXPIRY=7d  # Can be: 7d, 604800, etc.
```

### Example Usage

```typescript
// Create token
const token = TokenService.sign(123, 'user');
// Returns: eyJhbGc...

// Verify token
const payload = TokenService.verify(token);
// Returns: { userId: 123, role: 'user', iat: ..., exp: ... }

// Invalid token throws error
TokenService.verify('invalid.token.here');
// Throws: TokenExpiredError or JsonWebTokenError
```

### Testing Approach

Tests should cover:
1. Token creation with different users/roles
2. Token verification with valid tokens
3. Token expiration (mock time if needed)
4. Invalid token formats
5. Tampered tokens
6. Missing JWT_SECRET environment variable

---

## Notes for Implementer

- Start by examining existing JWT patterns in the codebase (if any)
- Use industry-standard library (e.g., `jsonwebtoken` npm package)
- Ensure environment variables are properly loaded
- Add comprehensive error messages for debugging
- Keep token payload minimal (performance)
- Consider token refresh strategy (for future enhancement)

## Notes for Verifier

- Verify all criteria are testable and met
- Check error handling for edge cases
- Ensure types are strict (no `any`)
- Verify tests cover: success, expiry, invalid tokens, tampering
- Ensure environment variables are validated
- Check code follows project linting rules
