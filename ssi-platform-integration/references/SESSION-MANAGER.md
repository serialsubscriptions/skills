# SessionManager API Reference

Server-side session management. Handles token storage, automatic refresh, cookie management, and role-based authorization.

```typescript
import { SessionManager, sessionRoles } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

```typescript
// From createSSI() — preferred
const session = ssi.session(req);              // API routes (has Request)
const session = ssi.session(cookieHeader);     // from cookie string
const session = ssi.session();                 // no cookie context
const session = await ssi.sessionAsync();      // Next.js server components (auto-detects cookies)
```

## Properties

| Property | Type | Description |
|----------|------|-------------|
| `sessionId` | `string \| undefined` | Session ID extracted from cookies (read-only) |

## Methods

### `requireAuth(opts?): Promise<SessionAuthResult>`

Verifies session validity and optional role/org/claim requirements. Returns a discriminated union.

```typescript
interface SessionAuthRequirement {
  roles?: string[];
  requireOrganization?: boolean;
  requiredClaims?: Record<string, ClaimValidator>;
}

type SessionAuthResult =
  | { authorized: true; sessionId: string; claims: Record<string, unknown> }
  | { authorized: false; reason: 'no_session' | 'no_claims' | 'missing_role' | 'no_organization' | 'missing_claim'; status: 401 | 403; claim?: string };
```

**Examples:**

```typescript
// Login-only check
const auth = await session.requireAuth();

// Role check
const auth = await session.requireAuth({ roles: sessionRoles.userRoles });

// Organization required
const auth = await session.requireAuth({
  roles: sessionRoles.userRoles,
  requireOrganization: true,
});

// Custom claim validation
const auth = await session.requireAuth({
  requiredClaims: {
    organization_id: 'positive_integer',
    email: (v) => typeof v === 'string' && v.endsWith('@example.com'),
  },
});

// Handle result
if (!auth.authorized) {
  return Response.json({ ok: false, error: auth.reason }, { status: auth.status });
}
const { sessionId, claims } = auth;
```

### `setSession(...): Promise<{ sessionId: string; setCookieHeader: string }>`

Stores tokens and returns session ID + Set-Cookie header.

```typescript
// With AuthServer (after callback)
const { sessionId, setCookieHeader } = await session.setSession(auth, tokenResponse);
const { sessionId, setCookieHeader } = await session.setSession(auth);

// With explicit tokens
const { sessionId, setCookieHeader } = await session.setSession({
  id_token: '...',
  access_token: '...',
  refresh_token: '...',
  refreshTtlSeconds: 604800,
  accessTokenTtlSeconds: 3600,
});
```

### `getSessionData(sessionId, dataKey): Promise<KVValue | null>`

Retrieves session data. Automatically refreshes tokens if < 60s remaining.

```typescript
type SessionDataKey = 'claims' | 'id_token' | 'access_token' | 'refresh_token';

const claims = await session.getSessionData(sessionId, 'claims');
const accessToken = await session.getSessionData(sessionId, 'access_token');
```

### `getClaim(claimName): Promise<unknown>`

Gets a specific claim from the session's ID token. Requires `sessionId` to be set.

```typescript
const userId = await session.getClaim('user_id');
const email = await session.getClaim('email');
```

### `hasRole(roleName): Promise<boolean>`

Checks if session has a specific role. Requires `sessionId`.

```typescript
if (await session.hasRole('admin')) { ... }
```

### `hasRoleOneOf(roleNames): Promise<boolean>`

Checks if session has at least one of the roles. Requires `sessionId`.

```typescript
if (await session.hasRoleOneOf(sessionRoles.adminRoles)) { ... }
```

### `clearSession(sessionId): Promise<string>`

Deletes all session data and returns a Set-Cookie header to clear the cookie.

```typescript
const deleteCookieHeader = await session.clearSession(session.sessionId!);
response.headers.set('Set-Cookie', deleteCookieHeader);
```

### `getSessionTtlSeconds(sessionId): Promise<number | null>`

Returns remaining TTL in seconds for the session's ID token.

### `generateSessionId(): string`

Generates a cryptographically strong, URL-safe session ID (43 chars, base64url).

### `buildSessionCookie(sessionId, overrides?): string`

Builds a Set-Cookie header string with configurable options.

```typescript
interface CookieOptions {
  name: string;
  domain?: string;
  path: string;
  sameSite: 'Lax' | 'Strict' | 'None';
  secure: boolean;
  httpOnly: boolean;
  maxAge: number;
}
```

## `sessionRoles` Constant

```typescript
export const sessionRoles = {
  allRoles:      ['platform_admin', 'member', 'owner', 'admin', 'billing', 'readonly'],
  adminRoles:    ['platform_admin', 'owner', 'admin'],
  userRoles:     ['member', 'owner', 'admin', 'billing', 'readonly'],
  userAdminRoles: ['platform_admin', 'owner', 'admin'],
};
```

## `ClaimValidator` Type

```typescript
type ClaimValidator =
  | 'present'           // not undefined or null
  | 'positive_integer'  // integer > 0 (string values parsed)
  | 'non_empty_string'  // non-empty string
  | ((value: unknown) => boolean);  // custom predicate
```

## Automatic Token Refresh

- Triggers when ID token TTL < 60 seconds
- Coordinated across instances to prevent duplicate refreshes
- Request-scoped: evaluates freshness at most once per instance
- On failure, methods return `null`

## Best Practices

- One `SessionManager` per request — never reuse across requests
- Use `requireAuth()` instead of manual `hasRole()` + `getClaim()` chains
- Use `sessionRoles` constants instead of hardcoding role arrays
- Always check for `null` returns from `getSessionData()` and `getClaim()`
- Trust automatic token refresh — don't implement manual refresh logic
- Always include `Set-Cookie` headers in responses after `setSession()` or `clearSession()`
