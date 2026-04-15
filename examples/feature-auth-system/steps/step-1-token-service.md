---
step_number: 1
name: Create Token Generation Service
created: 2026-04-08
---

# Step 1: Create Token Generation Service

## Goal

Create a reusable `TokenService` that generates JWT tokens with proper structure,
expiration, and verification. This service is the foundation for all authentication
in the system.

## Acceptance Criteria

Each criterion must be testable and independently verifiable.

- **Token Creation:** `TokenService.sign(userId, role)` creates valid JWT tokens with HS256 algorithm
- **Token Payload:** Token payload includes `userId` and `role` fields, verified via `TokenService.decode()`
- **Token Expiration:** Tokens expire after 7 days; `TokenService.verify()` throws error on expired tokens
- **Token Verification:** `TokenService.verify(token)` successfully verifies valid tokens and returns correct payload
- **Invalid Token Handling:** `TokenService.verify()` throws error for invalid or tampered tokens
- **Environment Config:** JWT_SECRET loaded from environment; JWT_EXPIRY configurable (default: 7d)
- **Type Safety:** All TypeScript types strict (no `any` types); full type definitions present
- **Unit Tests:** All unit tests pass, covering: creation, verification, expiry, invalid tokens, tampering
- **Build Success:** No TypeScript compilation errors; project builds successfully
- **Code Quality:** Code follows project conventions; linter (eslint) passes

## Implementation

### Files to Modify/Create

[To be filled by implementer during execution]

### Implementation Notes

[Summary of changes, test results, and any implementation decisions will be added by implementer]

## Verification

### Verification Notes

[Build results, test results, criterion verification evidence, and any issues found will be added by verifier]

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
