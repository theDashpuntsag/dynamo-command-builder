# Dynamo Command Builder

Type-safe builders for AWS SDK v3 DocumentClient command inputs.

This package helps you build correct DynamoDB command payloads with less boilerplate, safer defaults, and consistent placeholder handling.

It is especially useful when you want to:

- keep request-building logic out of handlers/controllers
- normalize query params from APIs into Dynamo query input
- avoid common expression mistakes in update/query/filter payloads

## Who This Is For

- Backend developers using `@aws-sdk/lib-dynamodb`.
- Agents and automation scripts that need deterministic Dynamo input generation.
- Teams that want reusable, tested request builders instead of hand-built expression strings.

## Install

This package expects AWS SDK v3 Dynamo packages as peer dependencies.

```bash
pnpm add dynamo-command-builder @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

or

```bash
npm i dynamo-command-builder @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

## Quick Start

```ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, QueryCommand } from '@aws-sdk/lib-dynamodb';
import { buildQueryCommandInput } from 'dynamo-command-builder';

const client = new DynamoDBClient({});
const doc = DynamoDBDocumentClient.from(client);

const input = buildQueryCommandInput({
  tableName: 'Users',
  queryRequest: {
    pKey: 'org#1',
    pKeyType: 'S',
    pKeyProp: 'pk',
    sKey: 'user#',
    sKeyType: 'S',
    sKeyProp: 'sk',
    skComparator: 'BEGINS_WITH',
    skValue2: 'user#',
    sorting: 'DESC',
    limit: 20,
  },
  filterExpression: '#status = :status',
  extraExpAttributeNames: { '#status': 'status' },
  extraExpAttributeValues: { ':status': 'ACTIVE' },
  projectionExpression: 'pk, sk, name, status',
});

const result = await doc.send(new QueryCommand(input));
```

## Exported API

### Command input builders

- `buildGetCommandInput`
- `buildPutCommandInput`
- `buildQueryCommandInput`
- `buildScanCommandInput`
- `buildUpdateCommandInput`

### Filter helpers

- `generateDynamoFilterAttributes`

### Query-request helpers

- `extractQueryReqFromParams`
- `createKeyConditionExpression`

### Error type

- `QueryParseError`

### Types

Includes strongly typed shapes such as:

- `CustomGetCommandInput`
- `CustomPutCommandInput`
- `CustomQueryCommandInput`
- `CustomScanCommandInput`
- `CustomUpdateCommandInput`
- `DynamoQueryRequest`
- `DynamoFilter`

## Recipes

### 1) Get item with projection and reserved keyword safety

```ts
import { GetCommand } from '@aws-sdk/lib-dynamodb';
import { buildGetCommandInput } from 'dynamo-command-builder';

const getInput = buildGetCommandInput({
  tableName: 'Users',
  key: { userId: 'u-123' },
  projectionExpression: 'name, size, #customField',
  expressionAttributeNames: { '#customField': 'customField' },
  consistentRead: true,
});

// projectionExpression becomes "#name, #size, #customField" when keywords are reserved
await doc.send(new GetCommand(getInput));
```

### 2) Put with condition expression

```ts
import { PutCommand } from '@aws-sdk/lib-dynamodb';
import { buildPutCommandInput } from 'dynamo-command-builder';

const putInput = buildPutCommandInput({
  tableName: 'Users',
  item: { userId: 'u-123', email: 'a@b.com', version: 1 },
  conditionExpression: 'attribute_not_exists(userId) OR #v = :v',
  expressionAttributeNames: { '#v': 'version' },
  expressionAttributeValues: { ':v': 1 },
  returnValues: 'NONE',
});

await doc.send(new PutCommand(putInput));
```

### 3) Query with key comparator, filter, and pagination

```ts
import { QueryCommand } from '@aws-sdk/lib-dynamodb';
import { buildQueryCommandInput } from 'dynamo-command-builder';

const queryInput = buildQueryCommandInput({
  tableName: 'Orders',
  queryRequest: {
    pKey: 'customer#42',
    pKeyType: 'S',
    pKeyProp: 'pk',
    sKey: '2026-01-01',
    sKeyType: 'S',
    sKeyProp: 'orderDate',
    skComparator: 'BETWEEN',
    skValue2: '2026-12-31',
    limit: 25,
    sorting: 'DESC',
    lastEvaluatedKey: { pk: 'customer#42', orderDate: '2026-06-01' },
  },
  filterExpression: '#status = :status',
  extraExpAttributeNames: { '#status': 'status' },
  extraExpAttributeValues: { ':status': 'PAID' },
  projectionExpression: 'pk, orderDate, total, status',
});

const page = await doc.send(new QueryCommand(queryInput));
```

### 4) Scan with manual filter expression

```ts
import { ScanCommand } from '@aws-sdk/lib-dynamodb';
import { buildScanCommandInput } from 'dynamo-command-builder';

const scanInput = buildScanCommandInput({
  tableName: 'Products',
  filterExpression: '#price BETWEEN :min AND :max',
  expressionAttributeNames: { '#price': 'price' },
  expressionAttributeValues: { ':min': 10, ':max': 100 },
  projectionExpression: 'productId, name, price',
  limit: 100,
});

await doc.send(new ScanCommand(scanInput));
```

### 5) Update with generated update expression

```ts
import { UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { buildUpdateCommandInput } from 'dynamo-command-builder';

const updateInput = buildUpdateCommandInput({
  tableName: 'Users',
  key: { userId: 'u-123' },
  item: {
    name: 'New Name',
    size: 'XL',
  },
  // If reserved words appear in generated SET expression, they are auto-aliased.
  conditionExpression: 'attribute_exists(userId)',
  returnValues: 'ALL_NEW',
});

const updated = await doc.send(new UpdateCommand(updateInput));
```

### 6) Update with explicit update expression

```ts
const explicitUpdate = buildUpdateCommandInput({
  tableName: 'Counters',
  key: { id: 'daily' },
  updateExpression: 'SET #count = #count + :inc',
  expressionAttributeNames: { '#count': 'count' },
  expressionAttributeValues: { ':inc': 1 },
  returnValues: 'UPDATED_NEW',
});
```

## Use This Package Well (Agent Guide)

If you are an agent generating data-access code, follow this checklist:

1. Use builder functions to create SDK inputs.
2. Keep table/index names and raw business params outside the builder.
3. Prefer `buildQueryCommandInput` over scan when key-based access is possible.
4. Use `extractQueryReqFromParams` at HTTP boundaries to normalize and validate query params.
5. Pass `lastEvaluatedKey` forward exactly as received for pagination.
6. For updates, choose exactly one strategy:
   - provide `updateExpression`
   - or provide `item` and let this package generate `SET ...`
7. For filters:
   - use `generateDynamoFilterAttributes` when you want declarative filter arrays
   - use manual `filterExpression` only when necessary
8. Catch `QueryParseError` for input/validation mistakes and return safe `400` responses.

## Query Param Normalization

`extractQueryReqFromParams` is designed for HTTP query strings and default query templates.

It can:

- normalize `index` and `indexName` into `indexName`
- coerce `limit` into a positive integer
- parse `lastEvaluatedKey` from object or stringified JSON
- enforce sort-key comparator requirements (`BETWEEN` requires `skValue2`, etc.)

Example:

```ts
import { extractQueryReqFromParams } from 'dynamo-command-builder';

const defaults = {
  indexName: 'USERS',
  pKey: 'DEFAULT_USER',
  pKeyType: 'S',
  pKeyProp: 'userId',
  limit: 20,
} as const;

const queryRequest = extractQueryReqFromParams(event.queryStringParameters, defaults);
```

## Filter DSL

`generateDynamoFilterAttributes` converts filter objects into:

- `filterExpression`
- `expressionAttributeNames`
- `expressionAttributeValues`

Example:

```ts
import { generateDynamoFilterAttributes } from 'dynamo-command-builder';

const generated = generateDynamoFilterAttributes([
  { name: 'age', condition: 'GREATER_THAN', type: 'N', value: '18' },
  { name: 'title', condition: 'CONTAINS', type: 'S', value: 'engineer' },
]);

// generated.filterExpression:
// "(#f0 > :f0v) AND (contains(#f1, :f1v))"
```

Supported conditions:

- `EQUAL_TO`, `NOT_EQUAL_TO`
- `LESS_THAN`, `LESS_THAN_OR_EQUAL_TO`
- `GREATER_THAN`, `GREATER_THAN_OR_EQUAL_TO`
- `BETWEEN`
- `BEGINS_WITH`
- `CONTAINS`, `NOT_CONTAINS`
- `EXISTS`, `NOT_EXISTS`

## Reserved Keywords

The package auto-handles reserved keywords in key places:

- projection expressions in get/query
- generated update expressions

Example:

- `size, name` can become `#size, #name`

You can still provide additional `ExpressionAttributeNames`; user-provided aliases override generated ones when keys collide.

## Errors and Validation

- `QueryParseError` is thrown for invalid request shapes in parser/scan paths.
- Zod schemas validate command input shapes and query/filter contracts.
- `buildUpdateCommandInput` throws if both are missing:
  - `updateExpression`
  - `item` with at least one field

## Known Behavior Notes (Current Version)

1. `buildScanCommandInput` currently works best with manual `filterExpression`; `filtersAttributes` should be treated as experimental in this version.
2. `buildUpdateCommandInput` only auto-aliases reserved words when it generates expression from `item`; if you pass `updateExpression`, pass your aliases yourself.
3. `buildQueryCommandInput` does not currently consume `keyConditionExpression` from custom input; key condition is generated from `queryRequest`.

## Minimal End-to-End Pattern

```ts
import { extractQueryReqFromParams, buildQueryCommandInput, QueryParseError } from 'dynamo-command-builder';
import { QueryCommand } from '@aws-sdk/lib-dynamodb';

export async function listUsers(event: { queryStringParameters: Record<string, string> | null }) {
  try {
    const queryRequest = extractQueryReqFromParams(event.queryStringParameters, {
      indexName: 'USERS',
      pKey: 'ORG#1',
      pKeyType: 'S',
      pKeyProp: 'pk',
      limit: 20,
    });

    const input = buildQueryCommandInput({
      tableName: 'MyTable',
      queryRequest,
      projectionExpression: 'pk, sk, name, status',
    });

    return await doc.send(new QueryCommand(input));
  } catch (err) {
    if (err instanceof QueryParseError) {
      return {
        statusCode: err.statusCode,
        body: JSON.stringify({ message: err.message }),
      };
    }
    throw err;
  }
}
```

## Local Development

```bash
pnpm test
pnpm lint
pnpm build
```

## License

MIT
