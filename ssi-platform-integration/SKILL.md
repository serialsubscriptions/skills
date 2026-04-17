---
name: ssi-platform-integration
description: >-
  Integrate the @serialsubscriptions/platform-integration npm package for
  session management, authentication, subscription plan management, usage
  reporting, and caching. Use when adding SSI auth, sessions, plans, limits,
  usage tracking, or the createSSI factory to a Next.js or Node.js project.
license: MIT
compatibility: Requires Node.js 18+, Next.js 15+, and Redis (production) or memory backend (dev)
metadata:
  author: serialsubscriptions
  package: "@serialsubscriptions/platform-integration"
---

# SSI Platform Integration

Integrate `@serialsubscriptions/platform-integration` into a Next.js or Node.js backend. The package provides session management, OAuth2/OIDC auth, subscription plan hydration, usage metering, and a caching layer.

## Install

```bash
npm install @serialsubscriptions/platform-integration
```

Peer dependency: `next@^15.0.0`. Storage backends are optional -- install only what you need:
- **Redis** (sessions, caching, storage): `npm install ioredis`
- **Postgres** (storage only): `npm install pg`
- **Memory** backend works out of the box with no extra dependencies.

## Core Concept: `createSSI()`

All components are created from a single factory. Create one shared module per project:

```typescript
// src/lib/ssi.ts
import { createSSI } from '@serialsubscriptions/platform-integration';

export const ssi = createSSI({
  issuerBaseUrl: process.env.SSI_ISSUER_BASE_URL!,
  clientId:      process.env.SSI_CLIENT_ID!,
  clientSecret:  process.env.SSI_CLIENT_SECRET!,
  redirectUri:   process.env.SSI_REDIRECT_URI!,
  storage: { backend: 'redis', container: 'ssi_storage', url: process.env.REDIS_URL! },
  cache:   { backend: 'redis', container: 'ssi',         url: process.env.REDIS_URL! },
});
```

The `ssi` object exposes these factory methods:

| Method | Returns | Purpose |
|--------|---------|---------|
| `ssi.session(req)` | `SessionManager` | Server-side session from a Request |
| `ssi.sessionAsync()` | `Promise<SessionManager>` | Session in Next.js server components (auto-detects cookies) |
| `ssi.auth()` | `AuthServer` | OAuth2/OIDC login, callback, logout |
| `ssi.plans(domain)` | `SubscribedPlanManager` | Fetch/hydrate subscription plans, features, limits |
| `ssi.usage(domain, token)` | `UsageApi` | Report and query subscription usage events |
| `ssi.cache` | `SSICache` | Namespaced cache (Redis or memory) |

## Required Environment Variables

```bash
SSI_ISSUER_BASE_URL=https://auth.example.com
SSI_CLIENT_ID=your-client-id
SSI_CLIENT_SECRET=your-client-secret
SSI_REDIRECT_URI=https://yourapp.com/api/v1/auth/callback
REDIS_URL=redis://localhost:6379
NEXT_PUBLIC_BASE_URL=https://yourapp.com   # for SessionClient (frontend)
```

See [references/SETUP.md](references/SETUP.md) for the full env var list, storage/cache backend options, and advanced config.

## Auth Routes (Next.js App Router)

Create `app/api/v1/auth/[...slug]/route.ts` to handle login, callback, logout, and session endpoints. The route template handles:

- `GET /api/v1/auth/login` — redirect to identity provider
- `GET /api/v1/auth/callback` — exchange code for tokens, set session cookie
- `GET /api/v1/auth/logout` — clear session, redirect to provider logout
- `GET /api/v1/auth/session` — return claims + TTL as JSON

See [references/SETUP.md](references/SETUP.md) for the complete route handler template.

## SessionManager (Server-Side)

Use in API route handlers where you have a `Request` object:

```typescript
import { ssi } from '@/src/lib/ssi';
import { sessionRoles } from '@serialsubscriptions/platform-integration';

export async function GET(req: Request) {
  const session = ssi.session(req);

  // Quick auth gate — returns discriminated union
  const auth = await session.requireAuth({
    roles: sessionRoles.userRoles,
  });
  if (!auth.authorized) {
    return Response.json({ ok: false, error: auth.reason }, { status: auth.status });
  }

  const { sessionId, claims } = auth;
  // ... use sessionId and claims
}
```

In Next.js server components (no `Request` available):

```typescript
const session = await ssi.sessionAsync();
```

Key methods: `requireAuth()`, `getSessionData()`, `getClaim()`, `hasRole()`, `hasRoleOneOf()`, `setSession()`, `clearSession()`.

See [references/SESSION-MANAGER.md](references/SESSION-MANAGER.md) for the complete API.

### Role Constants

```typescript
import { sessionRoles } from '@serialsubscriptions/platform-integration';

sessionRoles.adminRoles  // ['platform_admin', 'owner', 'admin']
sessionRoles.userRoles   // ['member', 'owner', 'admin', 'billing', 'readonly']
sessionRoles.allRoles    // all six roles
```

Always use `sessionRoles` constants instead of hardcoding role arrays.

## SessionClient (Frontend / Client-Side)

For browser components, server components that call the backend session endpoint, and middleware:

```typescript
import { SessionClient } from '@serialsubscriptions/platform-integration';

const client = SessionClient.getSessionClient();
if (await client.isLoggedIn()) {
  const orgName = await client.get('organization_name');
}
```

`requireAuth()` on the client is a UX guard — real authorization is enforced server-side:

```typescript
const auth = await client.requireAuth({
  redirectUrl: '/login',
  roles: ['admin', 'owner'],
});
if (!auth.authorized) router.push(auth.redirectUrl!);
```

See [references/SESSION-CLIENT.md](references/SESSION-CLIENT.md) for the complete API.

## SubscribedPlanManager

Fetch subscription plans with features and limits pre-attached:

```typescript
import { ssi } from '@/src/lib/ssi';

const session = ssi.session(req);
const accessToken = await session.getSessionData(session.sessionId!, 'access_token');

const plans = ssi.plans('https://account.example.com');
plans.setBearerToken(accessToken as string);

const allPlans = await plans.getAllPlans();
const plan = await plans.getPlanById(1);
const projectPlans = await plans.getPlansByProjectId(42);

// Limit helpers
if (plan) {
  const maxCalls = plans.getLimitMax(plan, 'api_calls');
  const enabled  = plans.isLimitEnabled(plan, 'api_calls');
  const overage  = plans.limitAllowsOverage(plan, 'api_calls');
}
```

**Important:** Always pass the `access_token` (not `id_token`) to `setBearerToken()`.

See [references/SUBSCRIBED-PLAN-MANAGER.md](references/SUBSCRIBED-PLAN-MANAGER.md) for the complete API.

## UsageApi

Report and query subscription usage events:

```typescript
import { ssi } from '@/src/lib/ssi';

const usage = ssi.usage('https://account.example.com', bearerToken);

// Single event
await usage.reportEvent({ limit_id: 123, amount: 1 });

// Batch (atomic transaction)
await usage.reportEvents([
  { limit_id: 123, amount: 1, project_id: 42, aggregate_id: GROUP_OR_SESSION_ID, aggregate_name: "Brand Acme Chat Usage" },
  { limit_id: 456, amount: 5, project_id: 42 },
]);

// Query usage
const allUsage = await usage.getUsageAll();
const limitUsage = await usage.getUsage(123);
const projectUsage = await usage.getProjectUsageAll(42);
```

See [references/USAGE-API.md](references/USAGE-API.md) for the complete API.

## SSICache

Namespaced caching with memory or Redis backends. Automatically configured by `createSSI()`:

```typescript
// Direct use
await ssi.cache.set('my:key', { data: 'value' }, 60);
const val = await ssi.cache.get<MyType>('my:key');

// Scoped prefixes
const jwksCache = ssi.cache.withPrefix('jwks');

// Compute-if-absent
const data = await ssi.cache.remember('expensive:op', 300, async () => {
  return await fetchExpensiveData();
});
```

See [references/CACHE.md](references/CACHE.md) for the complete API.

## Integration Checklist

When integrating `@serialsubscriptions/platform-integration` into a project:

1. `npm install @serialsubscriptions/platform-integration`
2. Set all required environment variables (see above)
3. Create `src/lib/ssi.ts` with `createSSI()` — export a singleton
4. Create `app/api/v1/auth/[...slug]/route.ts` using the auth route template
5. Use `ssi.session(req)` in API routes for auth checks
6. Use `SessionClient.getSessionClient()` on the frontend for session state
7. Use `ssi.plans(domain)` + `setBearerToken(accessToken)` for subscription data
8. Use `ssi.usage(domain, token)` for usage metering
9. Always use `sessionRoles` constants for role checks
10. One `SessionManager` per request — never share across requests

## Key Rules

- **One session per request**: `ssi.session(req)` binds to the current request's cookie. Never reuse across requests.
- **access_token for API calls**: `SessionManager` stores id_token, access_token, and refresh_token. Use `access_token` for `SubscribedPlanManager` and external APIs.
- **`requireAuth()` for route protection**: Prefer `session.requireAuth({ roles: ... })` over manual `hasRole()` + `getClaim()` chains. It returns a discriminated union with `status` codes.
- **Batch usage events**: Use `reportEvents([...])` over multiple `reportEvent()` calls — single HTTP request, atomic transaction.
- **Automatic token refresh**: `getSessionData()` and `getClaim()` automatically refresh tokens when < 60s remaining. Don't implement manual refresh.

## Additional References

- [Setup & Configuration](references/SETUP.md) — env vars, createSSI options, auth route template
- [SessionManager API](references/SESSION-MANAGER.md) — server-side session management
- [SessionClient API](references/SESSION-CLIENT.md) — frontend session client
- [SubscribedPlanManager API](references/SUBSCRIBED-PLAN-MANAGER.md) — plans, features, limits
- [UsageApi API](references/USAGE-API.md) — usage reporting and querying
- [SSICache API](references/CACHE.md) — caching layer
