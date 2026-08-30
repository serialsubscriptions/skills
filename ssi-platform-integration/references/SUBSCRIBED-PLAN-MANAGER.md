# SubscribedPlanManager API Reference

Fetches and hydrates subscription plans with features and limits from JSON:API endpoints.

```typescript
import { SubscribedPlanManager } from '@serialsubscriptions/platform-integration';
```

## Creating Instances

```typescript
// Via createSSI() — preferred (auto-wires cache)
const plans = ssi.plans('https://account.example.com');

// With custom API paths
const plans = ssi.plans(domain, {
  planApiBasePath: '/jsonapi/subscribed_plan/subscribed_plan',
  featureApiBasePath: '/jsonapi/subscribed_feature/subscribed_feature',
  limitApiBasePath: '/jsonapi/subscribed_limit/subscribed_limit',
  timeout: 5000,
});
```

**Always set the bearer token before API calls:**

```typescript
const accessToken = await session.getSessionData(sessionId, 'access_token');
plans.setBearerToken(accessToken as string);
```

Use the `access_token` from the session, **not** the `id_token`.

## Subscription status

`SUBSCRIPTION_STATUS.ACTIVE` (`active`) is a current subscription. `SUBSCRIPTION_STATUS.ARCHIVED` (`archived`) is inactive.

List helpers exclude archived plans unless you pass `{ includeArchived: true }`. `getPlanById` and `SSISubscribedPlanApi.list` are unfiltered.

## Plan Methods

### `getAllPlans(options?: ListPlansOptions): Promise<SubscribedPlan[]>`

Returns current plans with features and limits attached. Pass `{ includeArchived: true }` to include archived rows.

### `getPlanById(id): Promise<SubscribedPlan | null>`

Returns a single plan by internal numeric ID (`drupal_internal__id`). Not filtered by archive status.

### `getPlansByProjectId(projectId, options?: ListPlansOptions): Promise<SubscribedPlan[]>`

Returns current plans for a project. Pass `{ includeArchived: true }` to include archived rows.

### `getPlanForProject(projectId, options?: ListPlansOptions): Promise<SubscribedPlan | null>`

Returns the first current plan for a project, or `null`.

### `getPlansByUserId(userId, options?: ListPlansOptions): Promise<SubscribedPlan[]>`

Returns current plans for a user (internal user ID).

### `getPlanByProjectId(plans, projectId): SubscribedPlan[]`

Synchronous filter on already-fetched plans array.

## Limit Helper Methods

All accept `identifier` as either a limit name (string) or limit ID (number).

| Method | Returns | Description |
|--------|---------|-------------|
| `getLimitId(plan, name)` | `number \| null` | Internal numeric ID of a limit |
| `isLimitEnabled(plan, id)` | `boolean` | Whether limit status is true |
| `getLimitType(plan, id)` | `string \| null` | Limit type |
| `getLimitMax(plan, id)` | `number \| null` | Maximum value (`limit_value`) |
| `getLimitPeriod(plan, id)` | `string \| null` | Period (month, day, year) |
| `getLimitUnits(plan, id)` | `string \| null` | Unit name (singular) |
| `getLimitUnitsPlural(plan, id)` | `string \| null` | Unit name (plural) |
| `limitAllowsOverage(plan, id)` | `boolean` | Whether overage is allowed |
| `limitOverageMax(plan, id)` | `number` | Max overage (0 if not allowed) |

**Example:**

```typescript
const plan = await plans.getPlanById(1);
if (plan) {
  const maxCalls = plans.getLimitMax(plan, 'api_calls');
  const period   = plans.getLimitPeriod(plan, 'api_calls');
  const units    = plans.getLimitUnitsPlural(plan, 'api_calls');
  console.log(`${maxCalls} ${units} per ${period}`);

  if (plans.limitAllowsOverage(plan, 'api_calls')) {
    console.log(`Overage up to: ${plans.limitOverageMax(plan, 'api_calls')}`);
  }
}
```

## Types

### `SubscribedPlan`

```typescript
interface SubscribedPlan extends SubscribedPlanAttributes {
  uuid: string;
  features: SubscribedFeature[];
  limits: SubscribedLimit[];
}
```

### `SubscribedPlanAttributes`

Key fields:

| Field | Type | Description |
|-------|------|-------------|
| `drupal_internal__id` | `number` | Internal numeric ID |
| `name` | `string` | Plan name |
| `status` | `string` | Subscription status (e.g. `active`, `cancelled`, `trialing`) |
| `subscription_period` | `string` | Billing period |
| `organization_id` | `number` | Organization ID |
| `project_id` | `number \| null` | Project ID |
| `price_monthly` | `string` | Monthly price |
| `price_annually` | `string` | Annual price |
| `free_trial` | `boolean` | Whether plan has a free trial |
| `free_trial_days` | `number` | Trial duration |

### `SubscribedFeature`

```typescript
interface SubscribedFeature {
  uuid: string;
  drupal_internal__id: number;
  feature_name: string;
  feature_category: string;
  feature_type: string;
  feature_value: string | null;
  order: number;
}
```

### `SubscribedLimit`

```typescript
interface SubscribedLimit {
  uuid: string;
  drupal_internal__id: number;
  limit_name: string;
  limit_description: string;
  status: boolean;
  limit_units: string;
  limit_units_plural: string;
  limit_type: string;
  limit_value: number;
  limit_period: string;
  base_fee: string;
  limit_rate: number;
  allow_overage: boolean;
  overage_rate: number;
  overage_limit: number;
  order: number;
}
```

## Caching

Features and limits are cached for 24 hours (86400s) per plan ID. Plans themselves are fetched fresh.

Cache key format: `plan:{internalId}` with prefixes `features` and `limits`.

## Integration Pattern

```typescript
import { ssi } from '@/src/lib/ssi';

export async function GET(req: Request) {
  const session = ssi.session(req);
  if (!session.sessionId) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const accessToken = await session.getSessionData(session.sessionId, 'access_token');
  if (!accessToken) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const plans = ssi.plans('https://account.example.com');
  plans.setBearerToken(accessToken as string);

  const allPlans = await plans.getAllPlans();
  return Response.json(allPlans);
}
```

## Best Practices

- Always set bearer token with `access_token` before API calls
- Use `getPlansByProjectId()` instead of fetching all plans and filtering
- Use limit helper methods instead of manually searching `plan.limits`
- Handle `null` from `getPlanById()` — plan may not exist
- Access token must be fresh — `SessionManager` auto-refreshes tokens
