# UsageApi API Reference

Report and query subscription usage events.

```typescript
import { UsageApi } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

```typescript
// Via createSSI() — preferred
const usage = ssi.usage('https://account.example.com', bearerToken);

// Direct construction
const usage = new UsageApi('https://account.example.com', bearerToken);
```

## Reporting Methods

### `reportEvent(event): Promise<UsageApiResponse>`

Reports a single usage event.

```typescript
await usage.reportEvent({
  limit_id: 123,
  amount: 2.5,
  project_id: 456,
  metadata: { source: 'api', user_action: 'upload' },
});
```

### `reportEvents(events): Promise<UsageApiResponse>`

Reports multiple events atomically. All succeed or all fail (single transaction).

```typescript
await usage.reportEvents([
  { limit_id: 123, amount: 1, project_id: 456 },
  { limit_id: 123, amount: 3, project_id: 456 },
  { limit_id: 456, amount: 5, project_id: 456 },
]);
```

## Query Methods

### `getUsageAll(): Promise<UsageInfo[]>`

Returns usage for all limits.

```typescript
const allUsage = await usage.getUsageAll();
// [{ limit_id: 2, total_usage: 67.37 }, ...]
```

### `getUsage(limit_id): Promise<number | undefined>`

Returns total usage for a specific limit.

```typescript
const total = await usage.getUsage(123);
```

### `getProjectUsageAll(project_id): Promise<ProjectUsageInfo[]>`

Returns usage for all limits within a project.

```typescript
const projectUsage = await usage.getProjectUsageAll(456);
// [{ limit_id: 2, total_usage: 67.37, project_id: 456 }, ...]
```

### `getProjectUsage(project_id, limit_id): Promise<number | undefined>`

Returns usage for a specific limit within a project.

```typescript
const limitUsage = await usage.getProjectUsage(456, 123);
```

## Types

### `UsageEvent`

```typescript
interface UsageEvent {
  limit_id: number;              // required — limit entity ID
  amount?: number;               // default: 1.0
  project_id?: number;           // project entity ID
  aggregate_id?: number;         // grouping ID (campaign, job, etc.)
  aggregate_name?: string;       // human-readable aggregate name
  metadata?: Record<string, any> | string;  // arbitrary JSON
  event_timestamp?: number;      // Unix seconds UTC (default: server time)
}
```

### `UsageApiResponse`

```typescript
interface UsageApiResponse {
  status: string;  // "recorded" on success
  count: number;   // events processed
}
```

### `UsageApiError`

```typescript
interface UsageApiError {
  message: string;
  status?: number;       // HTTP status code
  statusText?: string;
}
```

### `UsageInfo`

```typescript
interface UsageInfo {
  limit_id: number;
  total_usage: number;
}
```

### `ProjectUsageInfo`

```typescript
interface ProjectUsageInfo {
  limit_id: number;
  total_usage: number;
  project_id: number;
}
```

## Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `{baseUrl}/api/v1/usage/report` | Report events |
| GET | `{baseUrl}/api/v1/usage/get` | Get all usage |
| POST | `{baseUrl}/api/v1/usage/get` | Get project usage (body: `{project_id}`) |

## Error Handling

```typescript
try {
  await usage.reportEvent({ limit_id: 123 });
} catch (error) {
  if (error.status === 401) {
    // Auth failed — check bearer token
  } else if (error.status === 400) {
    // Validation error — check limit_id, amount
  } else if (error.status >= 500) {
    // Server error — retryable
  } else {
    // Network error (no status)
  }
}
```

## Best Practices

- Use `reportEvents()` over multiple `reportEvent()` calls — single request, atomic transaction
- Provide `event_timestamp` for backdated events (Unix seconds UTC)
- Always check `events.length > 0` before calling `reportEvents()` — empty array throws
- Use `metadata` for debugging context (source, user_id, request_id)
- The bearer token is the OAuth access_token from the user's session
