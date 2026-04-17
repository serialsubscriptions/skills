# SessionClient API Reference

Client-side session management for browser components, Next.js server components, and middleware.

```typescript
import { SessionClient } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

```typescript
// Next.js server component (auto-detects cookies via next/headers)
const client = SessionClient.getSessionClient();

// With explicit base URL
const client = SessionClient.getSessionClient('https://api.example.com');

// Middleware or non-Next server (pass current request's cookies)
const client = SessionClient.getSessionClient(baseUrl, request.headers.get('cookie'));

// Direct construction
const client = new SessionClient('https://api.example.com', cookieHeader);
```

Requires `NEXT_PUBLIC_BASE_URL` env var if no `baseUrl` is passed.

## Methods

### `isLoggedIn(): Promise<boolean>`

Checks if the user has a valid session.

```typescript
if (await client.isLoggedIn()) {
  // authenticated
}
```

### `get(claimName): Promise<unknown>`

Retrieves a specific claim value.

```typescript
const orgName = await client.get('organization_name');
const userId = await client.get('user_id');
```

### `getClaim(claimName): Promise<unknown>`

Alias for `get()`. Consistent with `SessionManager.getClaim()`.

### `getAll(): Promise<unknown | null>`

Returns all claims or `null` if no session.

### `getTtl(): Promise<number | null>`

Returns session TTL in seconds.

### `warmup(): void`

Non-blocking prefetch of session data.

```typescript
client.warmup(); // fire and forget
// later calls to isLoggedIn()/get() may already have data
```

### `requireAuth(opts): Promise<AuthResult>`

UX guard for client-side route protection. Real authorization is enforced server-side.

```typescript
interface AuthRequirement {
  redirectUrl: string;              // where to redirect if unauthorized
  roles?: string[];                 // user must hold at least one
  requireOrganization?: boolean;    // require organization_id > 0
  requiredClaims?: Record<string, ClaimValidator>;
}

interface AuthResult {
  authorized: boolean;
  reason?: 'not_logged_in' | 'missing_role' | 'no_organization' | 'missing_claim';
  redirectUrl?: string;   // set when authorized === false
  claim?: string;         // set when reason === 'missing_claim'
}
```

**Browser (React client component):**

```typescript
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function AdminPage() {
  const router = useRouter();

  useEffect(() => {
    const client = SessionClient.getSessionClient();
    client.requireAuth({
      redirectUrl: '/login',
      roles: ['admin', 'owner'],
    }).then((result) => {
      if (!result.authorized) router.push(result.redirectUrl!);
    });
  }, [router]);

  return <div>Admin content</div>;
}
```

**Next.js server component:**

```typescript
import { redirect } from 'next/navigation';

export default async function AdminPage() {
  const client = SessionClient.getSessionClient();
  const auth = await client.requireAuth({
    redirectUrl: '/login',
    roles: ['admin', 'owner'],
  });
  if (!auth.authorized) redirect(auth.redirectUrl!);

  return <div>Admin content</div>;
}
```

## Session Endpoint

Communicates with `GET {baseUrl}/api/v1/auth/session`. Expects:

```json
{ "ok": true, "claims": { ... }, "ttl_seconds": 3600 }
```

## Cookie Handling

| Environment | Behavior |
|-------------|----------|
| Browser | `credentials: "include"` (automatic) |
| Next.js server component | Auto-reads via `next/headers` |
| Middleware / other server | Pass `cookieHeader` explicitly |

## Best Practices

- One client per request — `getSessionClient()` always returns a new instance
- Always `await` session methods before using results
- Use `warmup()` for background prefetch in middleware
- Validate claim types before use (`typeof orgName === 'string'`)
- `requireAuth()` is a UX helper — server-side `SessionManager.requireAuth()` enforces real authorization
