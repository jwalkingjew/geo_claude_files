# API Pagination

## Offset-based queries are capped

The `entities()`, `relations()`, and `values()` queries enforce `offset <= 1000`. Any request with `offset > 1000` returns a 400 error:

```
Pagination argument "offset" cannot exceed 1000
```

**Never use offset-based pagination for unbounded result sets.**

Offset-based queries are acceptable only for small bounded lookups (e.g., `entities(filter: { id: { in: [...] } })` where you control the batch size).

## Cursor-based connection endpoints

Use these for all paginated queries:

| Offset query | Connection query |
|-------------|-----------------|
| `entities(...)` | `entitiesConnection(...)` |
| `relations(...)` | `relationsConnection(...)` |
| `values(...)` | `valuesConnection(...)` |

### Pattern

**Important:** `entitiesConnection` takes `spaceId` and `typeId` as top-level arguments. `relationsConnection` and `valuesConnection` take a `filter` object instead.

```graphql
# entitiesConnection — top-level args
query {
  entitiesConnection(
    spaceId: "..."
    first: 500
    after: "cursor_string"   # omit on first request
  ) {
    nodes { id name }              # simpler access
    # or: edges { node { id name } }  # relay-style
    pageInfo { hasNextPage endCursor }
  }
}

# relationsConnection — uses filter object
query {
  relationsConnection(
    filter: { spaceId: { is: "..." }, typeId: { is: "..." } }
    first: 500
  ) {
    nodes { id fromEntityId toEntityId }
    pageInfo { hasNextPage endCursor }
  }
}
```

### TypeScript implementation

```typescript
async function fetchAll(spaceId: string): Promise<string[]> {
  const ids: string[] = [];
  let cursor: string | null = null;

  while (true) {
    const afterClause = cursor ? `after: "${cursor}"` : '';
    const data = await gql(`{
      entitiesConnection(spaceId: "${spaceId}" first: 500 ${afterClause}) {
        nodes { id }
        pageInfo { hasNextPage endCursor }
      }
    }`);

    const conn = data.entitiesConnection;
    for (const node of conn?.nodes ?? []) ids.push(node.id);
    if (!conn?.pageInfo?.hasNextPage) break;
    cursor = conn.pageInfo.endCursor;
  }

  return ids;
}
```

### Same pattern for relations and values

```typescript
// relationsConnection
let cursor: string | null = null;
while (true) {
  const afterClause = cursor ? `after: "${cursor}"` : '';
  const data = await gql(`{
    relationsConnection(
      filter: { spaceId: { is: "${spaceId}" }, fromEntityId: { in: [${filterIds}] } }
      first: 500
      ${afterClause}
    ) {
      nodes { id toEntityId toEntity { name } }
      pageInfo { hasNextPage endCursor }
    }
  }`);
  const conn = data.relationsConnection;
  // process conn.nodes ...
  if (!conn?.pageInfo?.hasNextPage) break;
  cursor = conn.pageInfo.endCursor;
}

// valuesConnection
let cursor: string | null = null;
while (true) {
  const afterClause = cursor ? `after: "${cursor}"` : '';
  const data = await gql(`{
    valuesConnection(
      filter: { propertyId: { in: [${filterIds}] } }
      first: 500
      ${afterClause}
    ) {
      nodes { propertyId spaceId text }
      pageInfo { hasNextPage endCursor }
    }
  }`);
  const conn = data.valuesConnection;
  // process conn.nodes ...
  if (!conn?.pageInfo?.hasNextPage) break;
  cursor = conn.pageInfo.endCursor;
}
```

### Parallel pagination with aliases

When paginating values and relations independently in the same query, use GraphQL aliases:

```graphql
{
  valConn: valuesConnection(filter: {...}, first: 500, after: "...") {
    nodes { propertyId spaceId }
    pageInfo { hasNextPage endCursor }
  }
  relConn: relationsConnection(filter: {...}, first: 500, after: "...") {
    nodes { typeId fromEntity { spaceIds } }
    pageInfo { hasNextPage endCursor }
  }
}
```

Track cursors independently (`valCursor`, `relCursor`) and stop each when `hasNextPage` is false.

## Connection filters

The connection endpoints support nested filters that can pre-filter results server-side:

```graphql
entitiesConnection(
  spaceId: "..."
  filter: {
    relations: {
      some: {
        spaceId: { is: "..." }    # IMPORTANT: always scope by space
        toEntity: {
          values: { none: { propertyId: { is: "..." }, text: { isNull: false } } }
          relations: { none: { typeId: { is: "..." }, toEntityId: { in: ["..."] } } }
        }
      }
    }
  }
)
```

### Filter gotchas

1. **Always include `spaceId` in relation filters.** Without it, the API scans across all spaces and will return `INTERNAL_SERVER_ERROR` on complex filters.

2. **Use `none` for exclusion logic, not `every`.** `every` means "ALL items must match the full condition" which fails when items have different field values. `none` means "NO item matches" which is the correct way to express "there is no X where Y".

   - Want "no name value exists": `values: { none: { propertyId: { is: NAME_ID }, text: { isNull: false } } }`
   - Want "not typed as Image": `relations: { none: { typeId: { is: TYPES_ID }, toEntityId: { in: [IMAGE_ID] } } }`

3. **Reduce `first` if filters are complex.** Start with 500; drop to 100 or 50 if you get internal server errors.
