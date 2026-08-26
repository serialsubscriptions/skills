# Setup & Configuration

## `createSSI()` Factory

All SSI components are created from a single factory call. Create one shared module:

```typescript
// src/lib/ssi.ts
import { createSSI } from '@serialsubscriptions/platform-integration';

export const ssi = createSSI({
  issuerBaseUrl: process.env.SSI_ISSUER_BASE_URL!,
  clientId:      process.env.SSI_CLIENT_ID!,
  clientSecret:  process.env.SSI_CLIENT_SECRET!,
  redirectUri:   process.env.SSI_REDIRECT_URI!,
  jwksPath:      process.env.SSI_ISSUER_JWKS_PATH,  // optional JWKS override
  storage: {
    backend: 'redis',
    container: 'ssi_storage',
    url: process.env.REDIS_URL!,
  },
  cache: {
    backend: 'redis',
    container: 'ssi',
    url: process.env.REDIS_URL!,
  },
});
```

### Lazy Singleton Pattern

For projects where config is loaded dynamically:

```typescript
import { createSSI } from '@serialsubscriptions/platform-integration';

let _ssi: ReturnType<typeof createSSI> | undefined;

export function getSSI() {
  if (_ssi) return _ssi;
  _ssi = createSSI({
    issuerBaseUrl: process.env.SSI_ISSUER_BASE_URL!,
    clientId:      process.env.SSI_CLIENT_ID!,
    clientSecret:  process.env.SSI_CLIENT_SECRET!,
    redirectUri:   `${process.env.NEXTAUTH_URL}/api/v1/auth/callback`,
    storage: buildStorageOpts(),
    cache:   buildCacheOpts(),
  });
  return _ssi;
}
```

### `createSSI()` Config Options

| Parameter | Required | Description |
|-----------|----------|-------------|
| `issuerBaseUrl` | Yes | OIDC issuer base URL. Typical origin for `SSIProjectApi` |
| `clientId` | Yes | OAuth2 client ID |
| `clientSecret` | Yes | OAuth2 client secret |
| `redirectUri` | No | OAuth2 redirect URI for callback |
| `jwksPath` | No | Custom JWKS endpoint path |
| `storage` | No | `StorageInitOpts` for session token storage |
| `cache` | No | `CacheInitOpts` for caching layer |

### `StorageInitOpts`

```typescript
interface StorageInitOpts {
  backend: 'redis' | 'postgres' | 'memory';
  container: string;
  url?: string;
  host?: string;
  port?: number;
  user?: string;
  password?: string;
  database?: string;
  ssl?: boolean;
}
```

### `CacheInitOpts`

```typescript
interface CacheInitOpts {
  backend?: 'redis' | 'memory';  // default: 'memory'
  container: string;
  url?: string;
  host?: string;   // default: 'localhost'
  port?: number;    // default: 6379
  user?: string;
  password?: string;
  tls?: boolean;
}
```

## Full Environment Variable Reference

### Auth (required)

| Variable | Description |
|----------|-------------|
| `SSI_ISSUER_BASE_URL` | OIDC issuer base URL. Also the typical origin for `SSIProjectApi` (and often for `ssi.plans()` / `ssi.usage()`) |
| `SSI_CLIENT_ID` | OAuth2 client ID |
| `SSI_CLIENT_SECRET` | OAuth2 client secret |
| `SSI_REDIRECT_URI` | OAuth2 redirect URI |
| `NEXTAUTH_URL` | App base URL (used to derive cookie domain) |

### Storage Backend

| Variable | Default | Description |
|----------|---------|-------------|
| `SSI_STORAGE_BACKEND` | — | `redis`, `postgres`, or `memory` |
| `SSI_STORAGE_CONTAINER` | `ssi_storage` | Storage key namespace |
| `SSI_STORAGE_URL` | — | Connection URL (alternative to host/port) |
| `SSI_STORAGE_HOST` | `localhost` | Host |
| `SSI_STORAGE_PORT` | `6379` | Port |
| `SSI_STORAGE_USER` | — | Username |
| `SSI_STORAGE_PASSWORD` | — | Password |
| `SSI_STORAGE_DATABASE` | — | Database name (postgres) |
| `SSI_STORAGE_SSL` | `false` | Enable TLS |

### Cache Backend

| Variable | Default | Description |
|----------|---------|-------------|
| `SSI_CACHE_BACKEND` | `memory` | `redis` or `memory` |
| `SSI_CACHE_CONTAINER` | `ssi` | Cache key namespace |
| `SSI_CACHE_URL` | — | Redis connection URL |
| `SSI_CACHE_HOST` | `localhost` | Redis host |
| `SSI_CACHE_PORT` | `6379` | Redis port |
| `SSI_CACHE_USER` | — | Redis username |
| `SSI_CACHE_PASSWORD` | — | Redis password |
| `SSI_CACHE_SSL` | `false` | Enable TLS |

### Cookie Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `SSI_COOKIE_NAME` | `ssi_session` | Session cookie name |
| `SSI_COOKIE_PATH` | `/` | Cookie path |
| `SSI_COOKIE_DOMAIN` | from `NEXTAUTH_URL` | Cookie domain |
| `SSI_COOKIE_SAMESITE` | `Lax` | `Lax`, `Strict`, or `None` |
| `SSI_COOKIE_SECURE` | `true` | HTTPS only |
| `SSI_SESSION_REFRESH_TTL_SECONDS` | `604800` (7 days) | Default refresh token TTL |

### Frontend

| Variable | Description |
|----------|-------------|
| `NEXT_PUBLIC_BASE_URL` | Backend API base URL for `SessionClient` |

## Auth Route Handler Template

Create `app/api/v1/auth/[...slug]/route.ts` in your Next.js App Router project. The file has two customization points at the top; everything else is generic:

```typescript
// app/api/v1/auth/[...slug]/route.ts
//
// SSI auth route template. Copy this file into any Next.js App Router project
// that uses @serialsubscriptions/platform-integration. The only things you need
// to customise are the two imports below.

// ---------------------------------------------------------------------------
// ▸ CUSTOMIZATION POINT 1: getSSI()
// ---------------------------------------------------------------------------
// Returns your project's createSSI() instance.
// Create a src/lib/ssi.ts that exports a singleton, then import it here.
// If you don't have a shared module yet, inline the config directly:
//
//    import { createSSI } from "@serialsubscriptions/platform-integration";
//    let _ssi: ReturnType<typeof createSSI>;
//    function getSSI() {
//      if (_ssi) return _ssi;
//      _ssi = createSSI({
//        issuerBaseUrl: process.env.SSI_ISSUER_BASE_URL!,
//        clientId:      process.env.SSI_CLIENT_ID!,
//        clientSecret:  process.env.SSI_CLIENT_SECRET!,
//        redirectUri:   process.env.SSI_REDIRECT_URI!,
//        storage: { backend: "redis", container: "ssi_storage", url: process.env.REDIS_URL! },
//        cache:   { backend: "redis", container: "ssi",         url: process.env.REDIS_URL! },
//      });
//      return _ssi;
//    }
//
import { getSSI } from '@/src/lib/ssi';

// ---------------------------------------------------------------------------
// ▸ CUSTOMIZATION POINT 2: getPostAuthRedirectUrl()
// ---------------------------------------------------------------------------
// Where to send the user after login/callback. Return your app's home URL.
function getPostAuthRedirectUrl(): string {
  return process.env.NEXT_PUBLIC_APP_URL ?? '/';
}

// ---------------------------------------------------------------------------
// ▸ GENERIC AUTH HANDLERS — nothing below here needs to change
// ---------------------------------------------------------------------------

export async function GET(req: Request) {
  const { pathname } = new URL(req.url);
  const segments = pathname.split('/').filter(Boolean);
  const action = segments[segments.length - 1];

  switch (action) {
    case 'callback': return handleCallback(req);
    case 'login':    return handleLogin(req);
    case 'logout':   return handleLogout(req);
    case 'session':  return handleSession(req);
    default:         return new Response('Not found', { status: 404 });
  }
}

async function handleCallback(req: Request) {
  const ssi = getSSI();
  const auth = ssi.auth();

  const { searchParams } = new URL(req.url);
  await auth.handleCallback({
    code: searchParams.get('code'),
    state: searchParams.get('state'),
  });

  const session = ssi.session(req.headers.get('cookie'));
  const { setCookieHeader } = await session.setSession(auth);

  return new Response(null, {
    status: 302,
    headers: {
      Location: getPostAuthRedirectUrl(),
      'Set-Cookie': setCookieHeader,
    },
  });
}

async function handleLogin(req: Request) {
  const ssi = getSSI();
  const session = ssi.session(req);

  if (session.sessionId) {
    const claims = await session.getSessionData(session.sessionId, 'claims');
    if (claims) return Response.redirect(getPostAuthRedirectUrl(), 302);
  }

  const auth = ssi.auth();
  const { url } = await auth.getLoginUrl({ stateTtlSeconds: 900 });
  return Response.redirect(url, 302);
}

async function handleLogout(req: Request) {
  const ssi = getSSI();
  const auth = ssi.auth();
  const session = ssi.session(req);

  let deleteCookieHeader: string | undefined;
  if (session.sessionId) {
    deleteCookieHeader = await session.clearSession(session.sessionId);
  }

  const { url } = auth.getLogoutUrl();
  const headers = new Headers({ Location: url });
  if (deleteCookieHeader) headers.append('Set-Cookie', deleteCookieHeader);
  return new Response(null, { status: 302, headers });
}

async function handleSession(req: Request) {
  const ssi = getSSI();
  const session = ssi.session(req);

  if (!session.sessionId) {
    return Response.json({ ok: false, error: 'Not authenticated' }, { status: 401 });
  }

  const claims = await session.getSessionData(session.sessionId, 'claims');
  if (!claims) {
    return Response.json({ ok: false, error: 'Invalid or expired session' }, { status: 401 });
  }

  const ttl = await session.getSessionTtlSeconds(session.sessionId);
  return Response.json({ ok: true, claims, ttl_seconds: ttl });
}
```

### Session Endpoint Contract

The `/api/v1/auth/session` endpoint must return this shape for `SessionClient` to work:

**Success (200):**
```json
{ "ok": true, "claims": { ... }, "ttl_seconds": 3600 }
```

**Error (401):**
```json
{ "ok": false, "error": "Not authenticated" }
```
