# SSIProjectApi API Reference

JSON:API client for Drupal **project** entities on the SSI platform (`/jsonapi/project/project`). This is not the consumer app's `GET`/`POST /api/v1/project` routes.

`SSIProjectApi` is **not** wrapped by `createSSI()`. There is no `ssi.projects()`. Construct it directly.

```typescript
import { SSIProjectApi } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

The constructor `domain` is the Drupal/JSON:API origin. **Typically the same as `SSI_ISSUER_BASE_URL`** (the OIDC issuer host).

```typescript
const client = new SSIProjectApi(process.env.SSI_ISSUER_BASE_URL!);

// Optional path / timeout overrides
const client = new SSIProjectApi(process.env.SSI_ISSUER_BASE_URL!, {
  apiBasePath: '/jsonapi/project/project', // default
  timeout: 30000,                          // default (ms)
});
```

Default API path: `/jsonapi/project/project`.

## Authentication

Always set the session **`access_token`**, not the `id_token`.

```typescript
const accessToken = await session.getSessionData(sessionId, 'access_token');
client.setBearerToken(accessToken as string);
```

Default OAuth scopes already include `view_project`, `create_project`, `update_project`, and `delete_project`.

Write operations (`POST` / `PATCH` / `DELETE`) fetch and cache a CSRF token from `{baseUrl}/session/token`. Call `clearCsrfToken()` if a cached token is rejected.

| Method | Returns | Description |
|--------|---------|-------------|
| `setBearerToken(token)` | `void` | Set OAuth/JWT bearer token |
| `getBearerToken()` | `string \| null` | Current bearer token |
| `clearCsrfToken()` | `void` | Drop cached CSRF token |

## IDs

`get`, `update`, and `delete` take the JSON:API **UUID**, not the numeric `project_id`.

Numeric `project_id` is for plans and usage (`getPlansByProjectId`, `getProjectUsageAll`), not these methods.

## Methods

### `list(options?): Promise<JsonApiDocument>`

List projects with optional filters, sort, pagination, includes, and sparse fieldsets. Pagination `limit` is capped at 50.

```typescript
const projects = await client.list({
  filters: { status: 'active' },
  sort: ['-created'],
  pagination: { offset: 0, limit: 10 },
  include: ['owner'],
});
```

Filter values may be a scalar or `{ operator, value }`:

```typescript
await client.list({
  filters: {
    status: 'active',
    created: { operator: '>', value: '2024-01-01' },
  },
});
```

### `get(uuid, options?): Promise<JsonApiDocument>`

Fetch one project by UUID.

```typescript
const project = await client.get('550e8400-e29b-41d4-a716-446655440000', {
  include: ['owner', 'tags'],
});
```

### `create(attributes, options?): Promise<JsonApiDocument>`

Create a project. Default resource type is `project--project`.

```typescript
const created = await client.create(
  { name: 'New Project' },
  {
    relationships: {
      user_id: { type: 'user--user', id: userUuid },
      user_owner: { type: 'user--user', id: userUuid },
      organization_owner: { type: 'organization--organization', id: organizationUuid },
    },
  }
);
```

The created entity UUID is `created.data.id` (when `data` is a single resource).

### `update(uuid, attributes, options?): Promise<JsonApiDocument>`

Partial update by UUID.

```typescript
await client.update('550e8400-e29b-41d4-a716-446655440000', { status: 'completed' });
```

### `delete(uuid): Promise<boolean>`

Delete by UUID. Success is HTTP 204 (`true`).

```typescript
await client.delete('550e8400-e29b-41d4-a716-446655440000');
```

## Types

### `ListOptions`

```typescript
interface ListOptions {
  filters?: FilterParams;
  sort?: string[];                 // prefix '-' for descending
  pagination?: { offset?: number; limit?: number };
  include?: string[];
  fields?: Record<string, string[]>;
}
```

### `GetOptions`

```typescript
interface GetOptions {
  include?: string[];
  fields?: Record<string, string[]>;
}
```

### `CreateOptions` / `UpdateOptions`

```typescript
interface CreateOptions {
  relationships?: Record<string, { type: string; id: string } | { type: string; id: string }[]>;
  resourceType?: string;           // default: 'project--project'
}
```

### `JsonApiDocument`

```typescript
interface JsonApiDocument {
  data: JsonApiResource | JsonApiResource[];
  included?: JsonApiResource[];
  links?: { self?: string; first?: string; last?: string; next?: string; prev?: string };
  meta?: { count?: number; [key: string]: unknown };
}
```

## Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `{baseUrl}/jsonapi/project/project` | List |
| GET | `{baseUrl}/jsonapi/project/project/{uuid}` | Get one |
| POST | `{baseUrl}/jsonapi/project/project` | Create |
| PATCH | `{baseUrl}/jsonapi/project/project/{uuid}` | Update |
| DELETE | `{baseUrl}/jsonapi/project/project/{uuid}` | Delete (204) |
| GET | `{baseUrl}/session/token` | CSRF token for writes |

## Best Practices

- Pass `process.env.SSI_ISSUER_BASE_URL` as the constructor domain unless the JSON:API origin is known to differ
- Use `access_token` from the session, never `id_token`
- Use UUID for get/update/delete; use numeric `project_id` only with plans and usage APIs
- Do not confuse this client with a consumer app's local `/api/v1/project` REST API
- One client instance per request is fine; always `setBearerToken` before calls
