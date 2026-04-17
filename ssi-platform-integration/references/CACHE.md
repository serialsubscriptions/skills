# SSICache API Reference

Namespaced caching with memory or Redis backends. Supports TTL, bulk operations, distributed locking, and compute-if-absent.

```typescript
import { SSICache } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

```typescript
// Automatically configured via createSSI()
const cache = ssi.cache;

// Standalone initialization
const cache = SSICache.init({
  backend: 'redis',
  container: 'ssi',
  url: process.env.REDIS_URL!,
});

// Memory backend (dev/testing)
const cache = SSICache.init({
  backend: 'memory',
  container: 'ssi',
});
```

## Core Methods

### `get<T>(key): Promise<T | null>`

```typescript
const user = await cache.get<User>('user:123');
```

### `set<T>(key, value, ttlSec): Promise<void>`

```typescript
await cache.set('user:123', { name: 'John' }, 3600);
```

### `del(key): Promise<void>`

```typescript
await cache.del('user:123');
```

### `remember<T>(key, ttlSec, loader): Promise<T>`

Compute-if-absent: returns cached value or runs loader and caches the result.

```typescript
const data = await cache.remember('expensive:op', 300, async () => {
  return await fetchExpensiveData();
});
```

### `ttl(key): Promise<number | null>`

Returns remaining TTL in seconds.

## Bulk Methods

### `mget<T>(keys): Promise<(T | null)[]>`

```typescript
const [u1, u2, u3] = await cache.mget<User>(['user:1', 'user:2', 'user:3']);
```

### `mset<T>(entries): Promise<void>`

```typescript
await cache.mset([
  { key: 'user:1', value: alice, ttlSec: 3600 },
  { key: 'user:2', value: bob, ttlSec: 3600 },
]);
```

### `incrby(key, by?, ttlSec?): Promise<number>`

Atomic increment. Initializes to 0 if key doesn't exist.

```typescript
const count = await cache.incrby('page:views', 1, 3600);
```

## Prefix Scoping

### `withPrefix(prefix): SSICache`

Creates a scoped cache instance sharing the same backend connection.

```typescript
const jwksCache = cache.withPrefix('jwks');
// Keys become: container:jwks:key
```

## Distributed Locking (Redis only)

### `acquireLock(key, ttlMs): Promise<string | null>`

Returns lock token if acquired, `null` otherwise.

### `releaseLock(key, token): Promise<void>`

Releases a previously acquired lock.

```typescript
const token = await cache.acquireLock('critical:op', 5000);
if (token) {
  try {
    await doCriticalWork();
  } finally {
    await cache.releaseLock('critical:op', token);
  }
}
```

## `close(): Promise<void>`

Closes the backend connection. Only the root instance (without prefix) can close.

## Key Namespacing

Format: `container:prefix:key`

| Config | Key Passed | Final Redis Key |
|--------|-----------|-----------------|
| container=`ssi`, no prefix | `user:123` | `ssi:_:user:123` |
| container=`ssi`, prefix=`jwks` | `issuer1` | `ssi:jwks:issuer1` |

## Backends

| Backend | Use Case | Locking |
|---------|----------|---------|
| `memory` | Dev/testing, single instance | Naive (not distributed) |
| `redis` | Production, multi-instance | True distributed (`SET NX PX`) |

## Best Practices

- Use `withPrefix()` for logical separation (e.g. `jwks`, `session`, `rl`)
- Use `remember()` for expensive operations
- Use `mget`/`mset` for bulk operations instead of loops
- Set appropriate TTLs based on data freshness needs
- Always handle lock acquisition failures
- Use TypeScript generics for type safety: `cache.get<MyType>(key)`
